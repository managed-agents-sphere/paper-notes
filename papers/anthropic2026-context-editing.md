---
title: "Context editing — Claude API Docs"
authors: [Anthropic]
affiliation: Anthropic
arxiv:
year: 2026
read_date: 2026-04-27
tags: [context-engineering, agent-architecture, prompt-cache, memory]
related: [anthropic2026-context-windows, anthropic2026-prompt-caching, anthropic2026-token-counting, anthropic2026-compaction, anthropic2026-context-stack, 2604.14228-dive-into-claude-code]
cited_by: []
verdict: "5 件套里**最便宜的 in-session 原语**：无 inference cost、lossless（如工具可重调）、surgical 不动 user/assistant；`clear_at_least` 是 cache-aware 设计的活化石——清得太少不如不清。"
rating: 5/5
---

# Context Editing

> **关于这份笔记**：context-editing 是 5 件套里**最被低估**的一个——很多人见到 "compaction" 就用 compaction，错过了便宜得多的 tool-result clearing。这篇重点讲：和 compaction 的本质区别（sub-transcript vs whole-transcript）、`clear_at_least` 这个字段为什么是 cache-aware 设计的精华、以及它和 memory tool 的强协同模式。

## 1. TL;DR

`context_management.edits` 字段允许两种 edit type：
- **`clear_tool_uses_20250919`** — 清 `tool_result` 块的内容（保留 `tool_use` 调用记录）
- **`clear_thinking_20251015`** — 清 extended thinking 块；如同时启用，**必须放 edits 数组首位**

可调字段：`trigger`（触发阈值）/ `keep`（最近保留几个）/ `clear_at_least`（一次至少清多少 token）/ `exclude_tools`（豁免列表，典型如 `["memory","web_search"]`）。Beta header `context-management-2025-06-27`。

性质：**lossless**（如果工具可重调）/ **mechanical**（无 inference cost）/ **sub-transcript**（user / assistant 文字、tool_use 记录都不动）。响应里返回 `context_management.applied_edits` 告诉你清了几个、释放多少 token——**5 件套里唯一这么 observable 的**。

### 说人话

> 当 agent 连续读了 50 个文件、跑了 50 次 grep，它的对话历史里塞满了又长又过时的 tool 输出——context-editing 就是**"老的 tool 结果删除，但是工具调用记录还留着"**。Agent 还知道它读过哪些文件、grep 了什么，只是看不到具体内容了——下次需要可以**直接再调**。

#### 术语翻译

| 原词 | 说人话 |
|---|---|
| **`clear_tool_uses_20250919`** | "清空 tool_result 内容"操作的官方名（带版本日期） |
| **`clear_thinking_20251015`** | "清空 extended thinking blocks"操作的版本名 |
| **`tool_use` block** | 模型说"我要调 X 工具"的记录 |
| **`tool_result` block** | 工具返回的实际内容（这个会被清） |
| **`trigger`** | 何时触发清理（input_tokens 数 / tool_uses 数超过阈值） |
| **`keep`** | 保留最近 N 个 tool result（旧的清掉） |
| **`clear_at_least`** | 一次清理**至少**释放多少 token——核心 cache-aware 字段 |
| **`exclude_tools`** | 豁免列表，被点名的工具的结果**不会**被清 |
| **`applied_edits`** | 响应里告诉你这次清了什么的字段 |

#### 关键直觉的类比

**context-editing = 整理桌面**：
- 你做 research，桌上堆了 30 本书（tool results）
- compaction 是"写一篇综述笔记，把书全部还回图书馆"——综合理解但失原文
- context-editing 是"把不常用的书放回书架，桌上只留最近 5 本"——**书还在书架（可以再借），桌面更干净**
- 还在桌上的书页：你写的 notes（user/assistant text）和"我借过 XX 书"的借阅记录（tool_use blocks）—— 这些不动

**为什么 `clear_at_least` 重要**：
- 想象你每次只把"1 本书放回书架"——意义不大，反而打断思路（=cache invalidation）
- 一次至少放 10 本回去——值得这次"打断"——这就是 `clear_at_least` 的精髓

#### "lossless if tool is re-callable" 什么意思

context-editing 删的是 *tool result 内容*，不是 *tool 知识*：
- ✓ 模型还知道它"读过 file X、搜过 query Y"
- ✓ 如需细节可以**重新调用工具**
- ✗ 当下不能凭记忆复述细节

所以"lossless" 仅在工具**可重调** 的前提下成立——读本地文件 OK，调有 rate limit 的远程 API 就**有代价**了（多花 quota）。

## 2. 论文身份

- **标题**：Context editing
- **作者**：Anthropic
- **出处**：Claude API Docs，build-with-claude 系列
- **链接**：<https://platform.claude.com/docs/en/build-with-claude/context-editing>
- **快照日期**：2026-04-27（区域屏蔽，通过 SDK + cookbook + Anthropic 博客 + GitHub openclaw 反馈交叉确认）

## 3. 它解决什么问题

**问题**：长程 agent 的 context 增长**主要来自工具结果**，不是对话本身。每次 read_file 30-100K token、每次 search 几 K token 都会累积进 context，且**之后每个 turn 都要传**——线性增长很快撞 [[anthropic2026-context-windows]] 上限。

**为什么 [[anthropic2026-compaction]] 不是最优解**：
- compaction 跑一次 LLM 摘要——**贵**
- compaction lossy——文档的具体细节会被压成"读过该文件"
- compaction 触发后 cache 全失效

**为什么 [[anthropic2026-prompt-caching]] 不是最优解**：
- caching 让重读 cheap，**不删** token——预算还是会爆

**Context editing 文档**回答的是：
- **怎么按 *字段类型* 选择性删除，而不是粗暴 truncate？** → `clear_tool_uses_20250919`
- **怎么保证模型还"知道它做过什么"？** → `tool_use` block 保留，仅清 `tool_result.content`
- **怎么不让 cache 频繁失效？** → `clear_at_least` 强制每次清够多
- **怎么和 memory tool 协同？** → `exclude_tools: ["memory"]`

## 4. API 机制 / 契约

### 4.1 完整字段示例

```python
from anthropic.types.beta.beta_context_management_config_param import (
    BetaContextManagementConfigParam,
)

config: BetaContextManagementConfigParam = {
    "edits": [
        {
            "type": "clear_tool_uses_20250919",
            "trigger": {"type": "input_tokens", "value": 30_000},
            "keep": {"type": "tool_uses", "value": 3},
            "clear_at_least": {"type": "input_tokens", "value": 5_000},
            "exclude_tools": ["web_search"],
        }
    ]
}

response = client.beta.messages.create(
    model="claude-sonnet-4-6",
    betas=["context-management-2025-06-27"],
    context_management=config,
    messages=[...],
    tools=[...],
)
```

### 4.2 字段语义

| 字段 | 类型 | 含义 |
|---|---|---|
| `type` | `clear_tool_uses_20250919` 或 `clear_thinking_20251015` | edit 类型 |
| `trigger` | `{"type":"input_tokens"\|"tool_uses", "value":N}` | 多大触发 |
| `keep` | `{"type":"tool_uses", "value":N}` | 保留最近 N 个 tool_result |
| `clear_at_least` | `{"type":"input_tokens", "value":N}` | **至少**释放 N tokens 才清；否则推迟 |
| `exclude_tools` | `string[]` | 不清这些 tool 的 result（按 tool name） |

**`trigger` vs `clear_at_least` 的区别**（关键）：
- `trigger`：决定"什么时候考虑清理"
- `clear_at_least`：决定"决定清理时至少清多少"——保护一次清得有意义

### 4.3 双 edit 协同（thinking + tool_result）

```python
"edits": [
    {"type": "clear_thinking_20251015", "trigger": {...}},   # 必须先
    {"type": "clear_tool_uses_20250919", "trigger": {...}},  # 再
]
```

> **关键**：`clear_thinking_20251015` 必须放 edits 数组**首位**（cookbook 明示）

### 4.4 响应字段（observable，唯一）

每次 messages.create 返回，如果触发了 clearing：

```python
response.context_management.applied_edits
# 形如：
# [{"edit_type": "clear_tool_uses_20250919",
#   "tool_uses_cleared": 7,
#   "tokens_cleared": 42_000}]
```

**这是 5 件套里唯一明确告诉你"刚才发生了什么 context 操作"的字段** —— compaction 也有 typed block 但需要客户端解析；clearing 直接告诉你数字。

### 4.5 与 memory tool 的关键协同

```python
"edits": [{
    "type": "clear_tool_uses_20250919",
    "trigger": {...},
    "keep": {...},
    "exclude_tools": ["memory"],   # ← critical
}]
```

不写 `exclude_tools: ["memory"]` 的后果：
- agent 调 memory tool 写入"用户偏好"
- clearing 触发，把 memory tool result 也清了
- agent 再读 memory 时**找不到自己刚写的**——无限重写循环

cookbook 文档把这个 pattern 列为**memory + clearing 联用必备配置**。

## 5. 可观测的行为（cookbook 实测）

### 5.1 Research agent baseline vs clearing

> 来自 `tool_use/context_engineering/context_engineering_tools.ipynb`

任务：8 篇 ~40K token 文档读 + synthesis。

| 配置 | Peak context | 完成？ | 触发次数 |
|---|---|---|---|
| Baseline（无 management） | ~340K | ✓（context rot） | - |
| `trigger=30K, keep=4, clear_at_least=10K` | <60K | ✓ | 多次 |

**关键观察**：每次 clearing 让 context "step-down"——不是平滑降，是阶梯式：清完掉一截、继续读、再爬升、再清。多次清理的曲线呈锯齿形。

### 5.2 与 baseline 的 file_read 数量对比

```
File reads: baseline=8, clearing=11
```

**含义**：clearing 触发后，agent 偶尔需要 *重读* 早些清掉的文件——但代价远小于 compaction 跑摘要。

### 5.3 三原语 all-three 联用 timeline

cookbook 最后给出 all three 的 trajectory：

```
turn  ctx=        markers
turn  1  ctx=  3,200
turn  3  ctx= 87,000  ◇ memory
turn  6  ctx=167,000
turn  8  ctx=200,000  ✂ CLEARING
turn  9  ctx=130,000  ◇ memory
turn 10  ctx=180,000  ⊟ COMPACTION
turn 11  ctx= 50,000
turn 12  ctx= 95,000
```

读这个 timeline 的方式：
- ✂ CLEARING 在 turn 8 把 200K 降到 130K——清掉早期 file_read 结果
- ⊟ COMPACTION 在 turn 10 把 180K 降到 50K——把上半段对话整体压成摘要
- ◇ memory 是 agent 主动调 memory tool 写入持久状态

**clearing 的"step"小但频繁；compaction 的"drop"大但稀少**——这是 cookbook 的核心 visualization。

### 5.4 `applied_edits` 字段实战

```python
for response in tool_runner:
    if response.context_management and response.context_management.applied_edits:
        for edit in response.context_management.applied_edits:
            print(f"Cleared {edit.tool_uses_cleared} tool_uses, "
                  f"freed {edit.tokens_cleared} tokens")
```

实测中可以看到具体清了哪几个 tool_use_id（按数量），帮 debug 阈值是否合理。

## 6. 关键数字与 gotcha

### 6.1 硬数字

| 字段 | 默认 / 限制 |
|---|---|
| Beta header | `context-management-2025-06-27` |
| `clear_thinking_20251015` 位置 | 必须 edits 数组首位（如启用） |
| 推理成本 | **0**（mechanical edit） |
| 触发判断时机 | server-side, prefill 后 |
| `exclude_tools` 配 memory 的必要性 | **必须**（否则 memory 会被自我清空） |
| `applied_edits` 可观测 | ✓（5 件套唯一） |

### 6.2 反直觉

- **clearing 不省 latency 多少**：tool_use 块还在，model 仍要 attention 它们——**主要省的是钱**（input cost）和**推迟撞 context limit**
- **`tool_use` block 保留是关键设计**：模型记得它做过什么，只是看不到结果——这让"重读 same file" 变成模型的自然选择，而不是被迫
- **clearing 改前缀让 prompt cache 失效**：和 compaction 一样——`clear_at_least` 就是为此存在
- **不能选择性清"某 tool 的某次结果"**：粒度是"所有 tool 的最后 keep 个之外"
- **`keep` 是 *按 tool 数*不是按 token 数**：keep=5 意味着保留最新 5 个 tool result——不管它们多大；3 个 100K 的 tool result 也只算 3 个
- **trigger 类型可以是 tool_uses**：`{"type":"tool_uses", "value":20}` 意为"满 20 次 tool 调用就触发"——某些 tool-heavy 工作流这比 token 计数更稳定

### 6.3 `clear_at_least` 的数学含义（cache-aware 精髓）

设：
- 当前 cached prefix = C tokens
- 一次 cache_creation 成本 ≈ 1.25 × C × base_input_price
- 一次 cache_read 成本 ≈ 0.1 × C × base_input_price
- 节省 = (next-call-tokens-saved) × base_input_price - cache rebuild cost

**为了让 clearing 净收益 > 0**：

```
next_call_tokens_saved >  1.15 × C
（等价：clear_at_least > C × 1.15）
```

意思：**清的量必须超过现有 cache 的 1.15 倍才划算**。所以：
- 如果你 cache 30K，`clear_at_least` 应该 ≥ 35K（不是 5K！）
- 否则每次清都让 cache 重建，反而亏

cookbook 默认 `clear_at_least=5K` 是因为 cache prefix 还小；生产中要按当前 cache 大小动态调

### 6.4 Gotcha 清单

1. **不要漏 `exclude_tools: ["memory"]`**——必接近第一条
2. **不要用 keep=1**：留太少导致 agent 立刻 re-fetch；keep=3-6 是合理区间
3. **不要 trigger=很低值**：每次都触发反而损 cache
4. **不要把 clear_thinking 放第二位**：cookbook 警告必须首位
5. **不要假设 lossless 适用所有 tool**：rate-limited / 计费 API 重调有真实代价
6. **不要遗忘 betas header**：少了 `context-management-2025-06-27` 配置会被静默忽略

## 7. 限制

**Anthropic 文档承认（隐式）**：
- 仅清 tool_result 和 thinking，**user / assistant 文字增长 clearing 帮不上**——必须靠 [[anthropic2026-compaction]]
- 模型 minor version 切换可能改变 tool_use 行为，clearing 配置可能要重调
- thinking edit type 还在 beta，可能会改

**我看出的**：
- **没有"预演"机制**：和 compaction 一样，`clear_at_least` 设错也得真触发后才知道
- **没有 callback hook**：清完以后想做点什么（如通知 user / 写日志）没有 hook
- **粒度限制**：不能"清掉某 tool 名下的 result"——要么 exclude 全部，要么不动；想细粒度只能客户端先包装 tool
- **`applied_edits` 字段没记录"哪次清的"**：多次 clearing 累计后只看到最近一次的数字
- **2-stage thinking + tool_result clearing 默认行为没说清**：cookbook 没给出"thinking 先清还是 tool 先清"的语义保证；阅读源码或测试

## 8. 与我关心的问题的联系（context assemble / compact / cache）

### 8.1 在 5 件套依赖图里

| 原语 | 操作对象 | 推理成本 | Cache 影响 | 适用场景 |
|---|---|---|---|---|
| context-windows | 总预算 | - | - | 元约束 |
| token-counting | 度量衡 | 0 | - | 决策辅助 |
| **context-editing** | **sub-transcript（仅 tool_result）** | **0** | **部分失效（按 clear_at_least 摊销）** | **tool-heavy agent** |
| compaction | whole transcript | 1 LLM | 全失效 | 长程对话 / 模型推理累积 |
| prompt-caching | prefix KV 复用 | 0 | - | 重复 prefix |

**含义**：context-editing **应该是默认开**——便宜、可观测、无副作用（在 tool 可重调前提下）。**先开 clearing，再考虑要不要加 compaction**。

### 8.2 与 [[anthropic2026-prompt-caching]] 的协同（critical 工程点）

> `clear_at_least` 是 5 件套里**唯一公开承认 cache-aware 设计**的字段。

正确配置法：
- 测出当前 session 的稳定 cache prefix 大小 C
- `clear_at_least.value` ≥ C × 1.15
- `trigger.value` ≥ C × 2 + 工作 workspace（保护 cache 至少能活到工作 batch 完成）

[[2604.14228-dive-into-claude-code]] 描述的 *microcompact* 把这个做到极致——**等 API 真返回 `cache_deleted_input_tokens` 才下手**，不是估算。

### 8.3 与 [[anthropic2026-compaction]] 的对比

```
                  compaction        context-editing
                  ───────────       ───────────────
推理成本           1 次 LLM           0
失败模式           lossy summary    lost (re-fetchable)  
操作对象           whole transcript  sub-transcript
触发开销           50K 起            灵活
适用              对话/思考累积       tool result 累积
```

**联用范式**（cookbook all-three）：
- clearing 设低 trigger（30K-100K），频繁清 tool result
- compaction 设高 trigger（180K+），偶发处理对话累积
- 两者**互不抵消**因为操作对象不同

### 8.4 与 memory tool 的协同（"clearing+memory" 是 production pattern）

```python
"edits": [{
    "type": "clear_tool_uses_20250919",
    "trigger": {...},
    "keep": {...},
    "clear_at_least": {...},
    "exclude_tools": ["memory"],   # 必须
}]
```

工作流：
1. agent 读文件 / 跑工具
2. 看到关键发现就 *主动* 写入 memory tool（持久化）
3. context 满了，clearing 把 tool_result 删掉
4. agent 还能读 memory（被 exclude），不丢已记录的发现
5. 跨 session 启动新对话，memory 还在

**这是 cookbook 推荐的"长程 agent"标准模式**。

### 8.5 工程化建议（managed-agent）

1. **默认开**：所有长 session 都加 `clear_tool_uses_20250919`，trigger 设到 (window × 0.3)
2. **`exclude_tools` 总配**：至少 `["memory"]`；如有 web_search 这种 rate-limited tool 也加
3. **监控 `applied_edits`**：每次清了多少 token、多少 tool_uses——校准 trigger
4. **trigger 用 input_tokens 而非 tool_uses**：除非你 tool 调用大小很均匀
5. **keep 不要过小**：3-6 范围；太小 agent re-fetch 频繁
6. **`clear_at_least` 按 cache 大小算**：见 §6.3 公式

### 8.6 不该套的场景

- **Tool 不可重调**：rate limit 严苛 / 调用有副作用（写操作）/ 不幂等——clearing 后重调有代价
- **审计场景**：tool result 被清后审计回溯困难——要么用 [[anthropic2026-context-editing]] + session log（双轨存原 tool result 到磁盘），要么不用
- **Tool result 都很小**：累积慢，不到 trigger，配了也没用——不如不开
- **不用 tool 的纯对话**：用 [[anthropic2026-compaction]]，clearing 没东西可清

### 8.7 一条容易被忽略的设计含义

> **`applied_edits` 字段是 5 件套里唯一明确告诉你"刚才发生了什么 context 操作"的 observable**

这意味着：
- 用 [[anthropic2026-compaction]] 时你看到一个 typed compaction block，要自己 parse
- 用 [[anthropic2026-prompt-caching]] 时只能从 `cache_read_input_tokens` 反推
- 但 context-editing **直接告诉你 cleared X tool_uses, freed Y tokens**

→ **生产监控 dashboard 第一个该接的就是 `applied_edits`**。

---

## 引用与相关

- 原文：<https://platform.claude.com/docs/en/build-with-claude/context-editing>（区域屏蔽）
- 镜像：<https://docs.claude.com/en/docs/build-with-claude/context-editing>
- Cookbook（GitHub 可拉）：[`tool_use/context_engineering/context_engineering_tools.ipynb`](https://github.com/anthropics/claude-cookbooks/blob/main/tool_use/context_engineering/context_engineering_tools.ipynb)（三原语对比，含完整字段）
- SDK 完整字段示例：[`anthropic-sdk-python/examples/memory/basic.py`](https://github.com/anthropics/anthropic-sdk-python/blob/main/examples/memory/basic.py) —— `BetaContextManagementConfigParam`
- Memory tool docs（强协同对象）：<https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool>
- 第三方深度解析：[Context Editing Looks Like a Feature. It's Actually a Garbage Collector Without Write Barriers](https://conikeec.substack.com/p/context-editing-looks-like-a-feature)
- Agno 集成：<https://docs.agno.com/models/providers/native/anthropic/usage/context-management>
- Anthropic blog：[Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- 同 stack：[[anthropic2026-context-windows]] [[anthropic2026-prompt-caching]] [[anthropic2026-token-counting]] [[anthropic2026-compaction]]
- 综述：[[anthropic2026-context-stack]]
- 实现侧对照：[[2604.14228-dive-into-claude-code]]（其中 microcompact ≈ context-editing 的 cache-aware 极致版本）
- 我从哪里看到引用：用户（context engineering 研究方向）
