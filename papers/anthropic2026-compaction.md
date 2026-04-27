---
title: "Compaction — Claude API Docs"
authors: [Anthropic]
affiliation: Anthropic
arxiv:
year: 2026
read_date: 2026-04-27
tags: [context-engineering, agent-architecture, prompt-cache]
related: [anthropic2026-context-windows, anthropic2026-prompt-caching, anthropic2026-token-counting, anthropic2026-context-editing, anthropic2026-context-stack, 2604.14228-dive-into-claude-code, 2501.03276-commer]
cited_by: []
verdict: "最重的 context 操作（要跑一次 LLM 摘要）；whole-transcript 扁平化 + 50K 下限决定它只能用于 long horizon、不能频繁触发——和 cache 失效成本是孪生约束。"
rating: 5/5
---

# Compaction

> **关于这份笔记**：Compaction 是 5 件套里**唯一会跑一次 LLM 推理**的原语；最 lossy、也最强（处理 *whole transcript*，不只是 tool result）。新 server-side `compact_20260112` 仅 Opus 4.6 / Sonnet 4.6 支持。这篇重点讲：服务端 vs SDK 端两种实现的差异、`instructions` 的"完全替换"陷阱、和与 [[anthropic2026-prompt-caching]] / [[anthropic2026-context-editing]] 的 protocol。

## 1. TL;DR

当 conversation 接近 context window 上限，compaction 让模型**生成一段摘要 + 替换之前的全部历史**继续跑。两条实现路径：
- **Server-side**（推荐 Opus 4.6+）：`context_management.edits` 里加 `{"type":"compact_20260112","trigger":{...}}`，beta header `compact-2026-01-12`。**最低 trigger 50K tokens**，仅 Opus 4.6 / Sonnet 4.6 支持
- **SDK-side**（旧模型 / 更便宜的摘要 model）：`compaction_control={"enabled":True, "context_token_threshold":N}`，由 `tool_runner` 自动管

输出包在 `<summary></summary>` 标签里，作为 typed `compaction` content block 嵌入。默认 prompt 公开，自定义靠 `instructions` —— **完全替换**默认 prompt（不是追加）。lossy by design：**high-level facts 通常保留，verbatim 细节常丢**。

### 说人话

> 当对话太长快爆 context 时，compaction 让模型自己写一段会议纪要，然后把之前的对话**全部扔了**——只留摘要继续聊。

#### 术语翻译

| 原词 | 说人话 |
|---|---|
| **compaction** | "总结之前的对话 + 替换"操作（行业里也叫 conversation summarization） |
| **whole-transcript** | 不挑剔——user / assistant / tool_use / tool_result / 之前的 compaction block 全卷进摘要 |
| **`compact_20260112`** | 新版 server-side compaction 的 edit type 名（2026-01-12 是发布日） |
| **`compaction_control`** | SDK 端等价 API（针对 `tool_runner`） |
| **`<summary></summary>`** | 模型生成摘要时被指示包裹的 XML 标签——不是 parse 用，是 prompt 引导信号 |
| **`instructions`** | 自定义摘要器 prompt 的字段；**完全替换**默认（重要！） |
| **trigger** | 触发阈值；server-side 最低 50K input_tokens |
| **lossy** | 有损——细节会丢 |

#### 关键直觉的类比

**compaction = 把会议录像换成会议纪要**：
- 录像（原对话历史）信息全，但下次开会要快进太久
- 纪要（compaction summary）抓要点，但**说错的具体话术、长 tool 输出表格**全丢
- 一旦换成纪要，**录像就**作废**了——你拿不回原 KV cache，也回放不了细节

**whole-transcript vs sub-transcript**（关键区分）：
- compaction 是"全部融化重塑"——user 的话、assistant 的回答、tool 调用都进同一个摘要
- [[anthropic2026-context-editing]] 的 tool clearing 是"仅删工具结果，user/assistant 不动"——所以 compaction 比 clearing **更激进、更 lossy**

#### "lossy by design" 什么意思

不是 bug，是设计：摘要本质就是有损压缩。cookbook（[[anthropic2026-context-stack]] 引用的 context_engineering_tools.ipynb）实测：
- **High-level facts 通常保留**：架构决定、关键数字、未解问题
- **Obscure specifics 通常丢失**：某 appendix 表格的某一格、agent 的某一句具体表述

> **重要工程结论**：要 high-fidelity 跨长程，**必须 compaction + memory tool 联用**——把 critical 细节存到 memory，再让 compaction 安全摘要其余部分。

## 2. 论文身份

- **标题**：Compaction
- **作者**：Anthropic
- **出处**：Claude API Docs，build-with-claude 系列
- **链接**：<https://platform.claude.com/docs/en/build-with-claude/compaction>
- **快照日期**：2026-04-27（区域屏蔽，通过 cookbook + Anthropic 博客 + 第三方实现交叉确认）

## 3. 它解决什么问题

**问题**：长程 agent / 多 turn chatbot 的 context 单调增长，最终撞 [[anthropic2026-context-windows]] 上限——或 *在撞上限前*已经因 *context rot* 召回下降。

**为什么 caching / clearing 单独不够**：
- [[anthropic2026-prompt-caching]] 让重读便宜，**不删 token**——预算还是会爆
- [[anthropic2026-context-editing]] 只能删 tool result，**user / assistant 文字不动**——长对话里很多增长来自模型自己的回答和用户提问，clearing 管不了
- 必须有一个**全 transcript 的 lossy 压缩**才能管所有增长源

**为什么必须 server-side**（旧 SDK-side 实现的痛点）：
- SDK-side compaction（`compaction_control`）必须在客户端做 token 计数 + 触发判断 + 自己拼摘要请求
- 服务端**已经知道每次 prefill 的 token 数**，让它自己触发更准确、更省一次 round-trip
- 服务端的 `compact_20260112` 是**typed content block**，正确处理 tool_use/tool_result pairing，避免摘要切断 tool 调用对

## 4. API 机制 / 契约

### 4.1 Server-side（推荐路径）

```python
response = client.beta.messages.create(
    model="claude-opus-4-6",
    max_tokens=4096,
    betas=["compact-2026-01-12"],            # ← beta header 必需
    context_management={
        "edits": [{
            "type": "compact_20260112",
            "trigger": {"type": "input_tokens", "value": 180_000}, 
            # 可选: "instructions": "...自定义摘要 prompt..."
        }]
    },
    messages=[...]
)
```

**关键约束**：
- 仅 **Opus 4.6 / Sonnet 4.6** 支持（其他 model 会 API error）
- `trigger.value` 最低 **50,000 tokens**（更低值返回 API error）
- 默认（不指定 trigger）会用 server 自适应阈值（具体不公开）

### 4.2 SDK-side（旧 model / 灵活摘要）

```python
runner = client.beta.messages.tool_runner(
    model="claude-sonnet-4",
    max_tokens=4096,
    tools=tools,
    messages=messages,
    compaction_control={
        "enabled": True,
        "context_token_threshold": 5000,         # 默认 100K
        "model": "claude-haiku-4-5",             # 可选，用更便宜 model 摘要
        "summary_prompt": "...",                  # 可选，自定义
    },
)
```

**关键约束**：
- 仅 `tool_runner` 抽象支持，不直接通过 messages.create
- 仅 SDK >= 0.74.1
- 不与 server-side web search / extended thinking 兼容（cache 累积导致过早触发）

### 4.3 输出格式（关键）

server-side 触发后，response 里会**插入一个 typed content block**：

```python
{"type": "compaction", "content": [{"type":"text", "text":"<summary>...</summary>"}]}
```

下一次请求里你**把这个 block 原样塞回 messages**，服务端会自动**丢弃此 block 之前的所有 messages**：

```python
next_request_messages = [
    {"role": "assistant", "content": [
        {"type": "compaction", "content": last_compaction_block.content}
    ]},
    {"role": "user", "content": "继续"}
]
```

### 4.4 默认 summarization prompt（公开）

cookbook 引用的官方默认 prompt 全文（值得记住）：

> You have written a partial transcript for the initial task above. Please write a summary of the transcript. The purpose of this summary is to provide continuity so you can continue to make progress towards solving the task in a future context, where the raw history above may not be accessible and will be replaced with this summary. Write down anything that would be helpful, including the state, next steps, learnings etc. You must wrap your summary in a `<summary></summary>` block.

### 4.5 `instructions` 是**完全替换**

```python
"edits": [{
    "type": "compact_20260112",
    "trigger": {"type": "input_tokens", "value": 150_000},
    "instructions": "Focus on preserving code snippets, variable names, and technical decisions."
}]
```

**重要陷阱**：你提供 `instructions` 时，默认 prompt 整段被替换——你必须**自己负责完整 framing**（包括"包在 `<summary>` 标签里"这种格式约束）。一句简短 hint 是不够的；最佳实践是**附加到默认 prompt 之上写**，而不是从零开始：

```python
"instructions": """[default prompt 全文 here]

Additionally, focus on preserving:
- Specific lifespan numbers per organism
- The exact text of the user's correction in turn 5
- Tool call counts per tool"""
```

## 5. 可观测的行为（cookbook 实测）

### 5.1 SDK-side：customer service ticket 工作流

> 来自 `tool_use/automatic-context-compaction.ipynb`

任务：5 张工单，每张 7 个 tool call。

| 配置 | 总 turns | 累积 input | 触发次数 |
|---|---|---|---|
| 无 compaction | 27 | ~150,000 | 0 |
| `compaction_control={enabled:True, threshold:5000}` | ~30 | ~79,000 | 2 |

**含义**：低 threshold（5K）适合"独立 entity 处理"型任务（每张 ticket 独立，不需要 cross-reference）。**实际节省 ~47% input tokens**。

### 5.2 Server-side：研究 agent

> 来自 `tool_use/context_engineering/context_engineering_tools.ipynb`

任务：8 篇 ~40K tokens 的研究文档，分两 batch 读 + cross-doc synthesis。

| 配置 | Peak context | 是否完成 |
|---|---|---|
| Baseline（1M window，无 management） | ~340K | ✓（但 context rot） |
| Baseline（200K window） | 200K | ✗（hit limit） |
| `compact_20260112` trigger=180K | ~210K（compaction 后回落） | ✓ |
| compaction + clearing + memory（all 3） | <200K | ✓（适合所有 model） |

**反复出现的发现**：compaction 触发**一次**让 context 从 230K 回落到 ~120K（具体值依摘要长度）。

### 5.3 Cookbook 给的 cost analysis

> "compaction... costs inference (the summarizer model runs)"

意思：**每次 compaction 触发 = 一次额外 LLM 推理**（按 ~10K 输出摘要，~current_input 全量输入计费）。所以**频繁触发不经济**——这就是 50K 下限的工程理由。

### 5.4 cookbook 实证 high-level vs obscure 保留率

> "HIGH-LEVEL FACTS (expected to survive)" vs "OBSCURE SPECIFICS (expected to be lost)" 的 probe

实测大致：
- ✓ 高层级事实保留：3/3
- ✗ 偶发细节保留：0/3 - 1/3

**含义**：默认 prompt 让模型按"任务连续性"取舍——会保留你**任务上需要的**，丢弃**任务上不需要的**。如果你需要细节保留，要么改 `instructions`，要么用 [[anthropic2026-context-editing]] / memory tool 替代/补充。

## 6. 关键数字与 gotcha

### 6.1 硬数字

| 字段 | Server-side | SDK-side |
|---|---|---|
| 最低 trigger | **50,000 input_tokens** | 无下限（太低会自循环） |
| 默认 trigger | server-adaptive | 100,000 |
| 模型支持 | Opus 4.6 / Sonnet 4.6 | 大多数（含 4.5 / 4） |
| Beta header | `compact-2026-01-12` | 无 |
| 推理成本 | 1 次 LLM call（用主 model） | 1 次（可指定更便宜 model） |
| Output | typed `compaction` content block | 替换 messages 数组 |

### 6.2 反直觉

- **compaction 不能频繁触发**：每次 = 一次 LLM 推理 + cache 失效——5K trigger 在 customer service 里 OK，是因为每张 ticket 独立；如果你的对话有 cross-turn dependency，5K 太低会丢上下文
- **compaction summary 本身也消耗 token**：~5-10K 是典型摘要大小；这意味着**首个 turn 摘要后 input 不会归零**
- **不能用 server-side compaction 配合 server-side web search / extended thinking**：cache 累积导致过早触发（cookbook 明示）
- **prior compaction blocks 会被卷入下次摘要**：默认 prompt 没保护早期摘要，所以**长会话连续 compaction 会层层稀释**最早的信息——和有损视频压缩反复转码效果类似
- **trigger 配 input_tokens 类型时算的是 *prefill* 后的 token**，不是 messages 字符长度

### 6.3 Gotcha 清单

1. **不要假设 instructions 会追加到默认 prompt**——它替换全部，必须自己负责完整 framing
2. **不要在 hot loop 里用 SDK-side compaction**：客户端 token 计数 + 触发判断有 latency；server-side 更优
3. **不要对 compaction 后的对话期望 cache hit**：摘要替换全部历史 = prefix 全变 = cache 全失效。**摘要后下次调用要立刻重建 cache**（[[anthropic2026-prompt-caching]]）
4. **不要用 compaction 持久化跨 session**：摘要是 session 内的 typed block，新 session 必须把整个 transcript（包括摘要）传回——**真正跨 session 用 memory tool**
5. **不要把 trigger 设到 ≥ context_window**：会从来不触发——常见错误是把 1M model 配 trigger=900K，模型已经 context rot

## 7. 限制

**Anthropic 文档承认**：
- "lossy by design"——specific numbers / verbatim phrasing 会丢
- 仅 Opus 4.6 / Sonnet 4.6（server-side）
- 与 server-side web search / extended thinking 不兼容
- 50K 下限"为防摘要触发摘要"

**我看出的**：
- **没有 dry-run 模式**：你不能"试试这次 compaction 会摘出什么"，必须真的触发——iteration cost 高
- **`<summary></summary>` 标签是**软约束**，模型偶尔不遵守**：cookbook 注释说 "tags aren't parsed, but are there to help guide the model"——意味着**生产代码不能 strict parse**
- **没有 partial compaction**：要么全摘要，要么不摘——想"只摘前一半"做不到（用 SDK-side 自己实现可以）
- **没有 callback hook**：和 Claude Code 的 PreCompact hook（[[2604.14228-dive-into-claude-code]]）相比，API 端没有"compaction 即将触发，先做点别的"的 hook

## 8. 与我关心的问题的联系（context assemble / compact / cache）

### 8.1 在 5 件套依赖图里的位置

compaction 是**最贵但最强**的优化原语。决策序：

```
1. context-windows 不够        → 必须做点什么
2. 还能 cache 吗?             → 不变 prefix 加 prompt-caching
3. 增长来自 tool_result 吗?    → context-editing clear_tool_uses
4. 增长来自对话本身吗?         → compaction（最重）
5. 跨 session 需要持久状态?     → memory tool
```

**反方向看**：要避免 compaction，就要先把 1-3 用满。

### 8.2 与 cache 的协同（critical 工程问题）

> compaction 触发 = cache prefix 100% 失效 = 下次 prompt 重建 cache。

**正确做法**（参考 Claude Code 的 microcompact 设计）：
- compaction 触发**频率要低**——50K 下限就是为此
- compaction 后**立即在新请求里重新设 cache_control**——不要等到第二次才开
- 计算 cache 重建成本 vs compaction 节省，选 trigger 阈值时**至少留 1024 token 余量**让 cache 重建有意义

### 8.3 与 context-editing 的对比与协同

| | Compaction | Context-editing |
|---|---|---|
| 操作对象 | whole transcript | sub-transcript（仅 tool_result） |
| 推理成本 | 1 次 LLM | 0 |
| 是否 lossy | 是（lossy by design） | 否（如果 tool 可重调） |
| 适合 | 对话累积、模型推理累积 | tool 输出累积 |
| Cache 影响 | 全失效 | 部分失效（按 clear_at_least 摊销） |

**最佳实践（cookbook 的 "all three together"）**：
- compaction trigger 设高（180K-200K）
- context-editing trigger 设低（30K-100K）
- 两者**交错触发**——clearing 平时降 tool_result 增长，compaction 偶发处理对话累积

### 8.4 与 ComMer 的对比（latent vs textual compaction）

[[2501.03276-commer]] 是**latent**（连续向量）压缩：风格类记忆压得好、事实记忆 destructive interference。compaction 是**textual**（自然语言摘要）：
- ✓ 没有 destructive interference 问题（事实可保留）
- ✗ 不能压到很小（latent 可压 32 token；textual 摘要至少千字）
- ✓ 模型直接读懂（latent 需要训练）
- ✗ 有损（重新表述会引入误差）

**结论**：textual compaction 是 *online* 操作（无需训练）、对所有 model 可用——这是 API 选 textual 路线的根本原因。

### 8.5 工程化建议（managed-agent）

1. **server-side first**：除非用旧 model 或要省钱用 Haiku 摘要，永远用 `compact_20260112`
2. **trigger 配置法**：从 (context_window × 0.6) 起步——给 cache 重建和正常对话留足空间
3. **`instructions` 的设计**：默认 prompt 是好起点；要细节保留，**追加**而不是替换；写完测 high-fidelity probe
4. **compaction 不是替代 memory**：跨 session 需求**永远走 memory tool**
5. **monitor**：每个 session 跟踪 compaction 触发次数、每次释放的 token 数；阈值调到"每 session ≤2 次"

### 8.6 不该用的场景

- **<50K total context tasks**：compaction 唯一不能配置——也用不上
- **强一致性审计场景**：摘要的 lossy 性质让审计回溯不可靠；要么用 [[anthropic2026-context-editing]]（lossless 可重调）+ session log，要么干脆不优化
- **需要 verbatim 保留的工作流**（法务 / 编辑工作流）：summary 会改写，verbatim 全失
- **Server-side web search / extended thinking 流**：兼容性 bug，等 Anthropic 修
- **<5 turn 短对话**：增加 latency 与不可预测性，得不偿失

---

## 引用与相关

- 原文：<https://platform.claude.com/docs/en/build-with-claude/compaction>（区域屏蔽）
- Anthropic blog：[Managing context on the Claude Developer Platform](https://claude.com/blog/context-management)；[Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- Cookbook（GitHub 可拉）：
  - SDK-side：[`tool_use/automatic-context-compaction.ipynb`](https://github.com/anthropics/claude-cookbooks/blob/main/tool_use/automatic-context-compaction.ipynb)
  - Server-side 三原语对比：[`tool_use/context_engineering/context_engineering_tools.ipynb`](https://github.com/anthropics/claude-cookbooks/blob/main/tool_use/context_engineering/context_engineering_tools.ipynb)
  - 手动 instant compaction：[`misc/session_memory_compaction.ipynb`](https://github.com/anthropics/claude-cookbooks/blob/main/misc/session_memory_compaction.ipynb)
- Opus 4.6 launch：[Introducing Claude Opus 4.6](https://www.anthropic.com/news/claude-opus-4-6)
- 第三方深度解析：[Context Compaction API Complete Guide — Claude Lab](https://claudelab.net/en/articles/api-sdk/compaction-api-context-management)
- Bedrock 等价：<https://docs.aws.amazon.com/bedrock/latest/userguide/claude-messages-compaction.html>
- 同 stack：[[anthropic2026-context-windows]] [[anthropic2026-prompt-caching]] [[anthropic2026-token-counting]] [[anthropic2026-context-editing]]
- 综述：[[anthropic2026-context-stack]]
- Latent 对照：[[2501.03276-commer]]（latent compaction 的边界）
- 实现侧对照（Claude Code 5 层 pipeline 的 auto-compact 层）：[[2604.14228-dive-into-claude-code]]
- 我从哪里看到引用：用户（context engineering 研究方向）
