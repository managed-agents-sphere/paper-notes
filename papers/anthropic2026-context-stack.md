---
title: "Anthropic Context Stack：从 token 预算到跨会话记忆的五件套（综述）"
authors: [Anthropic, Xianping (synthesis)]
affiliation: Anthropic
arxiv:
year: 2026
read_date: 2026-04-27
tags: [context-engineering, prompt-cache, memory, agent-architecture]
related: [anthropic2026-context-windows, anthropic2026-token-counting, anthropic2026-prompt-caching, anthropic2026-compaction, anthropic2026-context-editing, 2604.14228-dive-into-claude-code, 2501.03276-commer]
cited_by: []
verdict: "Anthropic 把 context engineering 拆成 5 个可独立配置的 API 原语。理解它们的依赖关系（caching 是地基，count_tokens 是度量衡，clearing/compaction/memory 各管一类增长源）比掌握任何单一原语都重要——选错组合，1M context window 也救不了你。"
rating: 5/5
---

# Anthropic Context Stack：从 token 预算到跨会话记忆的五件套

> **关于这篇综述**：用八段式不能装下要讲的东西，所以这篇不是论文笔记，是**综述（survey）**。它把 [[anthropic2026-context-windows]] / [[anthropic2026-token-counting]] / [[anthropic2026-prompt-caching]] / [[anthropic2026-compaction]] / [[anthropic2026-context-editing]] 这 5 篇逐篇笔记的内容**拼成一张工程图**——每个原语是什么、它们怎么互相依赖、什么场景该用哪几个组合。读完这篇就知道：5 件套不是 5 个独立工具，是**一个分层的 context 工程界面**。

## 1. TL;DR + 说人话

Anthropic 把"如何在有限 context 内完成长程任务"这件事，拆成了 5 个可独立配置的 API 原语：

1. **context-windows**：你买的 token 预算（1M / 200K，按 model）
2. **token-counting**：免费的度量衡（estimate）
3. **prompt-caching**：让重复 prefix 便宜 90%（5min/1h TTL）
4. **compaction**：触发 LLM 摘要，处理 *whole transcript* 增长（≥50K trigger，仅 4.6 系列）
5. **context-editing**：mechanical 删除 tool result（`clear_tool_uses_20250919`），处理 *sub-transcript* 增长

> **核心洞察**：5 个原语的关系是**分层依赖**——caching 是地基（让所有别的便宜）、count_tokens 是度量衡（决策依据）、clearing/compaction/memory 各管不同来源的 context 增长。**选错组合，1M context window 也救不了你**——配个昂贵 model 加全部 1M 也跑不完一个 tool-heavy 长程任务。

### 一句话每篇

| 篇 | 一句话 | 复杂度 |
|---|---|---|
| [[anthropic2026-context-windows]] | "你的 token 预算合同；1M 是天花板不是默认" | ★ |
| [[anthropic2026-token-counting]] | "免费的度量衡，但是 estimate 不能用于对账" | ★ |
| [[anthropic2026-prompt-caching]] | "最便宜的优化（read 0.1x），但 prefix 一变全废" | ★★ |
| [[anthropic2026-context-editing]] | "最被低估的原语；mechanical、lossless、sub-transcript" | ★★ |
| [[anthropic2026-compaction]] | "最重的（要跑 LLM）；whole-transcript、lossy、最低 50K 触发" | ★★★ |

### 说人话

> 5 件套就像**做饭**：
> - **context-windows** = 锅的大小（决定一次能煮多少菜）
> - **token-counting** = 厨房秤（先估再放）
> - **prompt-caching** = 提前洗好的常用配料（重复用快又省）
> - **context-editing** = 把吃完的盘子拿走（不动正在吃的）
> - **compaction** = 把剩饭打包成"昨天总结"（占地方少但失原味）

## 2. "论文身份"

这不是一篇 Anthropic 写的综述，是我**根据 5 篇官方文档 + cookbook + Anthropic 博客综合**写的。

- **底层素材**（5 篇逐篇 paper-note）：
  - <https://platform.claude.com/docs/en/build-with-claude/context-windows>
  - <https://platform.claude.com/docs/en/build-with-claude/token-counting>
  - <https://platform.claude.com/docs/en/build-with-claude/prompt-caching>
  - <https://platform.claude.com/docs/en/build-with-claude/compaction>
  - <https://platform.claude.com/docs/en/build-with-claude/context-editing>
- **方法论锚点**：[Anthropic Effective Context Engineering for AI Agents (2025)](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- **实证锚点**：[`tool_use/context_engineering/context_engineering_tools.ipynb`](https://github.com/anthropics/claude-cookbooks/blob/main/tool_use/context_engineering/context_engineering_tools.ipynb)（cookbook 三原语联用 baseline）

## 3. 问题与动机：为什么需要"context engineering"

LLM 推理的根本约束是**有限的 attention 预算**。这预算的具体表现就是 context window，但问题不止"撞天花板"：

- **硬天花板**：超过 model 的 context window，API 直接 `prompt_too_long`
- **软天花板**（[context rot](https://research.trychroma.com/context-rot)）：窗口越满，模型对早期内容的召回越差——**1M 模型不是免死金牌**
- **经济天花板**：每个 turn 都要 prefill 全部 context，**线性增长 = 线性涨价 + 线性涨延迟**

Anthropic 的 [Effective Context Engineering for AI Agents (2025)](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) blog 把这件事概括为：

> "context is finite with diminishing marginal returns, and the core discipline is finding the smallest set of high-signal tokens that maximize the likelihood of your desired outcome"

**"找到最小的 high-signal token 集合"** —— 这就是 context engineering 的字面定义。5 件套就是 Anthropic 给开发者的**5 把不同尺寸的镊子**。

## 4. 五原语速览表

| 原语 | 操作对象 | 性质 | 关键触发字段 | 主要成本 | 解决的问题 |
|---|---|---|---|---|---|
| **context-windows** | 总预算 | model-bound | （隐性）选 model | 算 token 单价 | 元约束 |
| **token-counting** | prompt 估值 | mechanical | 调用 `count_tokens` | 0（免费） | 决策辅助 |
| **prompt-caching** | prefix KV | server-side cache | `cache_control={ttl:5m\|1h}` | write 1.25x/2x；read 0.1x | 重复 prefix 太贵 |
| **context-editing** | sub-transcript（仅 tool_result + thinking） | mechanical | `trigger`/`keep`/`clear_at_least`/`exclude_tools` | 0（无推理）+ 工具重调 | tool-heavy 累积 |
| **compaction** | whole transcript | 1 LLM 推理 | `trigger.value ≥ 50K` + 可选 `instructions` | 1 次 LLM call + cache 全失效 | 对话+模型推理累积 |

> 还有第 6 个相关原语：**memory tool**（`memory_20250818`）——它跨 session，不属于 build-with-claude 这 5 篇 docs 的范畴，但是 cookbook 里和这 5 件套强协同（见 §7、§8）

## 5. 依赖图

5 个原语之间的依赖关系：

```
              ┌─────────────────────────────────────────────┐
              │   context-windows (model-bound, 1M / 200K)   │ ← 元约束
              └─────────────────────────────────────────────┘
                                  │
                                  │ 限制
                                  ▼
              ┌─────────────────────────────────────────────┐
              │   token-counting (free, estimate, RPM-limit) │ ← 度量衡
              └─────────────────────────────────────────────┘
                                  │
                  ┌───────────────┼───────────────┐
                  │               │               │
                  ▼               ▼               ▼
        ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
        │ prompt-caching│  │ context-edit │  │  compaction  │
        │ (read 0.1x)   │  │ (mechanical) │  │  (LLM call)  │
        │ 地基          │  │ sub-trans    │  │ whole-trans  │
        │ ──────────    │  │ ──────────   │  │ ──────────   │
        │ TTL 5m/1h     │  │ trig+keep+   │  │ trig ≥ 50K   │
        │ ≤4 breakpts   │  │ clear_at_lst │  │ Opus/Sonnet 4.6│
        └──────────────┘  └──────────────┘  └──────────────┘
              ▲                  │                 │
              │                  │ cache 部分失效   │ cache 全失效
              │     ◄────────────┴─────────────────┘
              │              （都改 prefix）
              │
        协同/对抗
```

**关键发现**：

1. **prompt-caching 是地基** —— 所有别的优化都建在 cache 之上，因为别的原语**改 prefix = cache 失效**
2. **token-counting 是度量衡** —— 没它，trigger 阈值是瞎设
3. **clearing 和 compaction 是孪生敌友** —— 它们都改 prefix（破 cache），但操作对象互补（sub-transcript vs whole-transcript），所以可以**联用**而不是取舍

## 6. 工作流食谱

按"任务复杂度 × context 增长源"分场景：

### 6.1 单轮长 context QA（"读这篇 200K 论文回答问题"）

**配置**：仅开 caching
```python
client.messages.create(
    cache_control={"type":"ephemeral", "ttl":"5m"},
    messages=[{"role":"user", "content":doc + "\n\n" + question}]
)
```

**理由**：
- 单轮无累积——不需要 clearing / compaction
- 多次问同一文档不同问题：cache 让第二次起 0.1x
- 5min TTL 自动续期足够覆盖会话

### 6.2 多 turn chatbot（"几十轮对话，无 tool"）

**配置**：caching + 偶发 compaction
```python
# 主循环
client.messages.create(
    cache_control={"type":"ephemeral", "ttl":"5m"},
    messages=conversation,
)

# 累积超 60% context_window 时
client.beta.messages.create(
    betas=["compact-2026-01-12"],
    context_management={"edits":[{
        "type":"compact_20260112",
        "trigger":{"type":"input_tokens","value":120_000}
    }]},
    messages=conversation,
)
```

**理由**：
- 没有 tool result 累积——clearing 没用武之地
- 对话本身和模型推理是增长源——只能 compaction
- compaction 触发频率低（120K 才触发）—— cache 失效成本可摊销

### 6.3 长程 agent（"连续读 50 个文件、调 100 次 tool"）

**配置**：caching + clearing + memory
```python
client.beta.messages.create(
    betas=["context-management-2025-06-27"],
    cache_control={"type":"ephemeral", "ttl":"1h"},   # 长程用 1h TTL
    context_management={
        "edits": [{
            "type":"clear_tool_uses_20250919",
            "trigger":{"type":"input_tokens","value":100_000},
            "keep":{"type":"tool_uses","value":5},
            "clear_at_least":{"type":"input_tokens","value":30_000},
            "exclude_tools":["memory"],          # ← critical
        }]
    },
    tools=[memory_tool, ...your_tools]
)
```

**理由**：
- 主增长源是 tool result —— clearing 直击
- 5 个最近 tool result 保留——agent 还有"近期感"
- `clear_at_least=30K` 远大于 cache prefix（典型 ~10K），保护 cache 重建摊销
- `exclude_tools=["memory"]` 防 memory 被自我清空
- 长 session 用 1h TTL（不付频繁 cache_creation 钱）

### 6.4 跨会话研究 agent（"分多次会话写一篇论文"）

**配置**：5 件全开 + memory tool
```python
client.beta.messages.create(
    betas=["context-management-2025-06-27", "compact-2026-01-12"],
    cache_control={"type":"ephemeral", "ttl":"1h"},
    context_management={
        "edits": [
            {"type":"clear_tool_uses_20250919",
             "trigger":{"type":"input_tokens","value":200_000},
             "keep":{"type":"tool_uses","value":6},
             "clear_at_least":{"type":"input_tokens","value":30_000},
             "exclude_tools":["memory"]},
            {"type":"compact_20260112",
             "trigger":{"type":"input_tokens","value":180_000}},
        ]
    },
    tools=[memory_tool, ...your_tools]
)
```

**理由**（参考 cookbook all-three-active）：
- clearing trigger 设高（200K），让 batch-1 的 reads 不被早期清掉
- compaction trigger 略低（180K），先于 clearing；事实是 compaction 处理 *对话* 累积、clearing 处理 *tool* 累积，互不抵消
- memory tool 跨 session 持久化关键发现
- 1h TTL 保护跨长 session 间隔

## 7. 与 Claude Code paper（[[2604.14228-dive-into-claude-code]]）的对照

刚写的 Claude Code 论文笔记描述了 Claude Code v2.1.88 实现的 **5 层 compaction pipeline**。把这 5 层和 5 件套 API 原语对照：

| Claude Code 层 | 等价 API 原语 | 备注 |
|---|---|---|
| Budget reduction（per-tool-result 限长） | （API 无） | 客户端 only：tool 结果超过 maxResultSizeChars 替换为 reference |
| Snip（older history trim） | （API 无） | 客户端 only：lightweight FIFO trim |
| Microcompact（cache-aware） | ≈ [[anthropic2026-context-editing]] `clear_tool_uses_20250919` | Claude Code 等 API 真返回 `cache_deleted_input_tokens` 才下手——比 API 默认的估算更精准 |
| Context collapse（read-time projection） | （**API 无**） | Claude Code 独家发明：原 history 不动，只改"读出来时呈现成什么样" |
| Auto-compact（LLM summary） | ≈ [[anthropic2026-compaction]] `compact_20260112` | Claude Code 用 PreCompact hook 让 user 注入指令；API 端用 `instructions` |

**关键发现**：

1. **5 件套覆盖 Claude Code 5 层中的 3 层**（caching 没列因为它是地基；context-editing ≈ microcompact；compaction ≈ auto-compact）
2. **2 层是 Claude Code 自建的**（budget / snip / collapse）—— 这些**不是 API 能直接给的能力**，必须客户端实现
3. **Claude Code 有的 API 没有的**：
   - `read-time projection`（"context collapse" 不写历史，只改 view）—— 实现 audit trail + lossy view 共存
   - hook 系统（PreCompact / PostToolUse）—— API 端只有静态 `instructions`
4. **API 有的 Claude Code 没用的**：
   - 1h TTL（Claude Code 走默认 5min）—— 可能是因为 Claude Code 内部 session 通常 <5min 就 hit 一次

**给 managed-agent 工程化的启发**：
- 5 件套 API 是**必要不充分**——实际产品级 agent 还要再补 budget reduction、snip、collapse 这些客户端层
- read-time projection 是 critical 设计：让"压缩"完全可逆——你的系统也应该考虑这种"原始数据不动，只改 view"的范式

## 8. 取舍矩阵

每个原语在 6 个维度上的得分：

| 原语 | 是否 lossy？ | 推理成本 | 影响 cache 吗 | 跨 session？ | observable？ | 复杂度 |
|---|---|---|---|---|---|---|
| context-windows | - | - | - | - | - | 0（被动） |
| token-counting | ⚠️ estimate | 0 | - | - | ✓（返回 input_tokens） | ★ |
| prompt-caching | ✗ no | 0 | **是地基** | ✗ | ✓ via cache_read_input_tokens | ★ |
| context-editing | ✗ no（如可重调） | 0 | 部分失效 | ✗ | ✓ via `applied_edits` | ★★ |
| compaction | ✓ lossy by design | **1 LLM call** | **全失效** | ✗ | ⚠️ typed block 自己 parse | ★★★ |
| memory tool（外援） | ✗ no（内容不变） | 工具调用开销 | （write 时改 prefix）| **✓** | ✓ tool 调用记录 | ★★ |

**怎么读这张表**：
- 只有 compaction 是有损 + 推理成本——所以是最后一招
- 只有 memory 跨 session——所以**长程 agent 必接 memory**
- context-editing 在所有维度上都"还好"——所以是被低估的**默认开**选项
- prompt-caching **看似简单（★）但效果最大**——一行 `cache_control` 节省 90%

## 9. "什么时候不要用 X"

每个原语都有不该开的场景，列在这里防止盲目套用：

### prompt-caching：不该开
- prefix 永远不重复（每次 retrieval 结果不同的 RAG）—— 100% miss，付 1.25x 写费用纯亏
- prefix < 1024 token（Sonnet）/ 4096（Opus、Haiku）—— 静默被忽略，配了等于没配
- 合规场景禁止 server-side caching（HIPAA / GDPR 严格场合）
- >1h 间隔的极低频调用 —— 5min 必然 miss、1h 也快到期，付的不值

### context-editing：不该开
- Tool 不可重调（rate limit 严苛 / 不幂等 / 写操作）
- Tool result 都很小、累积慢——不到 trigger 等于白配
- 不用 tool 的纯对话——没东西可清
- 审计场景需要 tool result 完整保留——双轨持久化代替

### compaction：不该开
- <50K total context tasks——配置不能（API error）也用不上
- 强一致审计场景——lossy 性质不允许
- 需要 verbatim 保留（法务、编辑工作流）—— summary 改写一切
- 同时用 server-side web search / extended thinking——cache 累积导致过早触发（cookbook 明示不兼容）
- <5 turn 短对话 —— 增加 latency 与不可预测性，得不偿失

### memory tool：不该开
- 用户面向 chatbot，每个对话独立 / 隐私强约束 —— 跨 session 持久化反而 bug
- session 短到不需要状态 —— 增加 tool 调用开销
- 严合规场景没本地存储能力 —— memory 是 client-side，要你自己实现

### token-counting：不该频繁调
- Hot loop 里 keystroke 级估值 —— 自己实现 char-based 近似，定期 ground truth
- 高 RPS 场景 —— 独立 rate limit 也会爆

## 10. 与我关心的问题的联系（managed-agent 工程化建议）

这一段是**最重要的**——读完前 9 节后，把它对应到我们的 managed-agent 实践：

### 10.1 默认配置模板（直接抄）

任何长程 agent 入口默认配：

```python
DEFAULT_CONTEXT_CONFIG = {
    "betas": ["context-management-2025-06-27", "compact-2026-01-12"],
    "cache_control": {"type": "ephemeral", "ttl": "5m"},  # 显式！防 2026-03 静默事件
    "context_management": {
        "edits": [
            {
                "type": "clear_tool_uses_20250919",
                "trigger": {"type": "input_tokens", "value": 100_000},
                "keep": {"type": "tool_uses", "value": 5},
                "clear_at_least": {"type": "input_tokens", "value": 30_000},
                "exclude_tools": ["memory"],   # always
            },
            # 可选：compaction 兜底（仅 4.6 系列）
            {
                "type": "compact_20260112",
                "trigger": {"type": "input_tokens", "value": 200_000},
            }
        ]
    }
}
```

### 10.2 监控 dashboard（生产必接）

至少接 4 个观测点：

1. **`cache_read_input_tokens / total_input` 命中率**：<60% 是异常信号
2. **`applied_edits.tokens_cleared` 累计**：每 session 多少 token 被 clearing 释放
3. **`compaction` block 出现次数**：高频意味着 trigger 设低、要调
4. **`usage.input_tokens vs count_tokens estimate` 偏差**：>2% 是 tokenizer 漂移信号

### 10.3 与 Claude Code 抄作业要点

[[2604.14228-dive-into-claude-code]] §8.2 列了 3 件 Claude Code 做对了：
- 渐进式（便宜的先）
- cache-aware（等真返回 cache_deleted 才下手）
- read-time projection（不动原 history）

5 件套 API 给了 1、2 但没给 3。我们做 managed-agent 时**第 3 条要自建**：
- compaction summary 与原 history 必须并存
- 可以做"compaction view" 隐藏在 UI 后，原 transcript 落盘
- resume / fork / rewind 必须能拿回原始数据

### 10.4 "选模型 = 选 context 策略"

这是 5 件套综合后最反直觉但最重要的结论：

| 模型 | 1M | server-side compaction | 默认应用策略 |
|---|---|---|---|
| Opus 4.7 | ✓ | ✗（仅 4.6） | caching + clearing |
| Opus 4.6 | ✓ | ✓ | 全 5 件套 |
| Sonnet 4.6 | ✓ | ✓ | 全 5 件套 |
| Sonnet 4 | ✓（tier 4，>200K 2x 价） | ✗ | caching + clearing；超 200K 路由到 4.6 |
| 更老 | 200K | ✗ | caching + SDK-side compaction |

**模型选型 ≠ 只看能力 / 价格**，要把"context 策略可用性"加进矩阵。

### 10.5 前瞻：还缺什么

5 件套 + memory tool 解决了大部分 in-session / cross-session 场景，但仍有缺口：

- **可逆性**：API 没 read-time projection；想做 audit 必须客户端落盘双轨
- **Hook**：compaction 没 PreCompact callback；要做"摘要前先存关键 fact"必须客户端代码做
- **细粒度**：context-editing 不能选择性删某 tool 的某次结果；要么 exclude 全部要么不动
- **跨账户共享 cache**：每个客户独立 cache，整体效率有上限
- **observable 不一致**：5 个原语的 telemetry 字段命名不统一（`cache_read_input_tokens` vs `applied_edits` vs `compaction` typed block），客户端要做适配层

这些是下一代 API 可能演进的方向，也是**managed-agent 平台需要自己补的**抽象层。

---

## 引用与相关

- 5 篇逐篇笔记（必读）：
  - [[anthropic2026-context-windows]]
  - [[anthropic2026-token-counting]]
  - [[anthropic2026-prompt-caching]]
  - [[anthropic2026-compaction]]
  - [[anthropic2026-context-editing]]
- Anthropic blog：
  - [Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) —— 整个 stack 的方法论锚点
  - [Managing context on the Claude Developer Platform](https://claude.com/blog/context-management) —— 平台视角
- Cookbook：
  - [`tool_use/context_engineering/context_engineering_tools.ipynb`](https://github.com/anthropics/claude-cookbooks/blob/main/tool_use/context_engineering/context_engineering_tools.ipynb) —— 三原语对比 + all-three-active 实测（cookbook 一等品）
  - [`tool_use/automatic-context-compaction.ipynb`](https://github.com/anthropics/claude-cookbooks/blob/main/tool_use/automatic-context-compaction.ipynb) —— SDK 端 compaction
  - [`misc/session_memory_compaction.ipynb`](https://github.com/anthropics/claude-cookbooks/blob/main/misc/session_memory_compaction.ipynb) —— 客户端 instant compaction 模式
  - [`misc/prompt_caching.ipynb`](https://github.com/anthropics/claude-cookbooks/blob/main/misc/prompt_caching.ipynb) —— caching 完整 demo
- SDK 字段权威：[`anthropic-sdk-python/examples/memory/basic.py`](https://github.com/anthropics/anthropic-sdk-python/blob/main/examples/memory/basic.py) —— `BetaContextManagementConfigParam`
- Memory tool docs：<https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool>
- 实现侧对照（必读）：[[2604.14228-dive-into-claude-code]] —— 5 层 compaction pipeline 与 5 件套 API 的映射
- Latent compaction 对照：[[2501.03276-commer]] —— 与 textual compaction 的边界区分
- Context rot 实证：[research.trychroma.com/context-rot](https://research.trychroma.com/context-rot)
- 我从哪里看到引用：用户研究方向（context assemble / compact / cache）；建议把这篇当成"读 5 篇 build-with-claude docs 的总结"
