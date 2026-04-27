---
title: "Prompt caching — Claude API Docs"
authors: [Anthropic]
affiliation: Anthropic
arxiv:
year: 2026
read_date: 2026-04-27
tags: [prompt-cache, context-engineering, agent-architecture]
related: [anthropic2026-context-windows, anthropic2026-token-counting, anthropic2026-compaction, anthropic2026-context-editing, anthropic2026-context-stack, 2604.14228-dive-into-claude-code]
cited_by: []
verdict: "最便宜的 context 优化（read 0.1x，省 90%），但 prefix 一变就失效；这条约束让上层一切 compaction/editing 设计都必须 cache-aware—— 不只是『我要不要清』，而是『我清得够不够多以摊销 cache 重建』。"
rating: 5/5
---

# Prompt Caching

> **关于这份笔记**：5 件套里**最赚钱**的一个原语——Anthropic 官方 cookbook 实测可降至少 80%（多 turn 场景）。但它的 footgun 也最多：默认 5 min TTL、prefix 一变全失效、最小 1024 token、2026 年 3 月还出过静默降级事件。这篇把数字、坑、和"上层原语必须配合 cache 设计"的工程含义一次讲清。

## 1. TL;DR

把请求里**不变的 prefix**（system prompt / 文档 / 工具定义 / 早期对话）标记 `cache_control={"type": "ephemeral"}`，第一次写入按 **1.25x base input** 收费（5 min TTL）或 **2x**（1h TTL），之后命中按 **0.1x**（90% 折扣）。最小可缓存 **1024 tokens（Sonnet）/ 4096（Opus、Haiku 4.5）**。两种用法：**自动**（`cache_control` 在 top-level，SDK 自动放 breakpoint 在最后可缓存 block）和 **显式**（per-block，up to **4 个 breakpoints**）。**最大 footgun**：prefix 一字之差就 100% miss，所以 [[anthropic2026-compaction]] 改前缀会让 cache 失效，[[anthropic2026-context-editing]] 的 `clear_at_least` 字段就是为了"清得够多以摊销 cache 重建"而存在。

### 说人话

> 你给 Claude 同一段长 prompt 多次（比如每次问一篇论文不同问题），第一次"写入缓存"贵 25%，之后所有重读便宜 90%。前提是**这段 prefix 一字不差**——改一个标点就全废。

#### 术语翻译

| 原词 | 说人话 |
|---|---|
| **prompt cache** | 服务器侧的 KV cache，前缀相同的请求复用之前算好的 attention KV |
| **`cache_control`** | API 字段，告诉服务器"这块前缀我会重复用，存一下" |
| **`ephemeral`** | 唯一支持的 cache type；意为"短期，到期自动清" |
| **breakpoint** | cache 块的边界标记；最多 4 个（每个独立 TTL） |
| **TTL** | Time-to-live，缓存存活时间。5 min 默认 / 1h 可选 |
| **`cache_creation_input_tokens`** | 这次请求**新写入**的 cache 大小（按 1.25x 或 2x 计费） |
| **`cache_read_input_tokens`** | 这次请求**命中**的 cache 大小（按 0.1x 计费） |

#### 关键直觉的类比

**prompt cache = 共享阅读笔记**：
- 你和朋友轮流读同一本书（prefix），第一次有人读完了写下笔记（cache_creation）
- 之后你们读那本书都先翻笔记（cache_read），只补读"今天新加的一章"（剩余 input）
- 但如果有人**改了书的开头一个字**——所有笔记作废，必须重新读

**为什么 prefix 必须严格相同**：KV cache 是按 token 序列存的，序列不同 KV 不同。这不是设计缺陷，是 transformer 的物理事实。

#### 反直觉点

**多 turn 对话开 caching 反而是 *自动* 模式最好**：很多人以为要手动管理 breakpoint，其实 auto 模式让 SDK 把 breakpoint **自动前移**——每次新 turn 都把最新 user 消息作为 breakpoint，旧的所有 prefix 都进 cache。**一行代码改动 = 全自动**。

**5 min TTL 不是限制，是设计决策**：高频调用会自动续期（每次 hit refresh），所以"会话内连续调用"几乎没有 TTL 焦虑。**1h TTL 才是给"会话间隙长"的特殊场景**——高 RPS 场景用 5min 反而更划算（不付 2x 升级费）。

## 2. 论文身份

- **标题**：Prompt caching
- **作者**：Anthropic
- **出处**：Claude API Docs，build-with-claude 系列
- **链接**：<https://platform.claude.com/docs/en/build-with-claude/prompt-caching>
- **快照日期**：2026-04-27（区域屏蔽，通过 cookbook + SDK + 第三方实测交叉确认）

## 3. 它解决什么问题

**问题**：Transformer 推理时 prefix 的 KV computation 是 attention 大头开销。同一段长 prefix 在多次调用里被重新算 N 次，是浪费。

**为什么 client-side caching 不够**：
- 你能 cache 自己的字符串，但**算 attention KV 的是服务器**——必须服务器侧 cache
- API gateway 没法 cache 模型推理中间状态
- "用 embedding 索引检索 + 拼 prompt" 是 RAG，不解决"重读同一段长 doc"问题

**Prompt caching 文档**回答的是：
- **怎么告诉服务器 "这段 prefix 我会重复用"？** → `cache_control` 字段
- **能 cache 多久？** → ephemeral 5min（自动续期）/ 1h（付费扩展）
- **多便宜？** → write 1.25x / read 0.1x（5min）；2x / 0.1x（1h）
- **能 cache 几个独立块？** → 4 个 breakpoints
- **最小多少 token 才值得 cache？** → 1024（Sonnet）/ 4096（Opus、Haiku 4.5）

## 4. API 机制 / 契约

### 4.1 两种用法

#### Auto caching（推荐）

```python
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=300,
    cache_control={"type": "ephemeral"},   # ← top-level
    messages=[
        {"role": "user", "content": LONG_PREFIX + user_question},
    ],
)
```

SDK 自动把 breakpoint 放在**最后一个可缓存 block**。多 turn 时**自动前移**。

#### Explicit breakpoints

```python
response = client.messages.create(
    model="...",
    messages=[{
        "role": "user",
        "content": [
            {"type": "text", "text": SYSTEM_DOCS, 
             "cache_control": {"type": "ephemeral"}},
            {"type": "text", "text": user_question}
        ]
    }],
)
```

**最多 4 个 breakpoints**（automatic caching 用掉 1 个 slot）。每个独立 TTL。

### 4.2 1h TTL（extended）

```python
"cache_control": {"type": "ephemeral", "ttl": "1h"}
```

- write 价 **2x base input**（vs 5min 的 1.25x）
- read 价相同 **0.1x base input**
- 用 `cache_creation` 子对象的 `ephemeral_5m_input_tokens` / `ephemeral_1h_input_tokens` 区分

### 4.3 Multi-turn 自动前移（关键模式）

cookbook `misc/prompt_caching.ipynb` 实测的 multi-turn 表：

| Request | Cache 行为 |
|---|---|
| Turn 1 | System + User:A 写入 cache |
| Turn 2 | System + User:A 命中；Asst:B + User:C 写入 |
| Turn 3 | System...User:C 命中；Asst:D + User:E 写入 |

**含义**：第二轮起，几乎 100% input tokens 都从 cache 来；只有"最新一句话 + 上一句的回答"按全价。

### 4.4 Usage 字段

每次 messages.create 响应里 `usage` 对象有：

```python
usage.input_tokens                  # 真实新输入 token
usage.output_tokens                 # 输出 token
usage.cache_creation_input_tokens   # 新写入 cache 的 token（按 1.25x 或 2x）
usage.cache_read_input_tokens       # 命中的 cache token（按 0.1x）
usage.cache_creation                # 子对象（开了 1h TTL 时存在）
usage.cache_creation.ephemeral_5m_input_tokens
usage.cache_creation.ephemeral_1h_input_tokens
```

### 4.5 cache 命中条件（严格）

要命中前缀必须**逐字节相同**。常见命中失败：
- prepend 一个时间戳到 user message → 永远 miss（cookbook 用这个手段做 baseline 对照）
- system prompt 改了一句 → 整个 cache 重建
- tool schema 变化 → 整个 cache 重建
- 模型 minor version 切换（如 sonnet-4-6 → 4-7）→ 不共享 cache

## 5. 可观测的行为（cookbook 实测）

### 5.1 单轮加 cache

cookbook `misc/prompt_caching.ipynb` 跑 *Pride and Prejudice*（~187K tokens）作大 context：

| 场景 | 时间 | input_tokens | cache_create | cache_read |
|---|---|---|---|---|
| 无 cache | ~14s | 187,000 | 0 | 0 |
| 第一次开 cache（写入） | ~14s | ~50 | ~187,000 | 0 |
| 第二次同 prompt（命中） | ~2.5s | ~50 | 0 | ~187,000 |

**关键发现**：
- 第一次的 *延迟* 与无 cache 相近（要先算 KV 再存）
- 第二次延迟降 **>5x**（attention prefix 跳过）
- 第二次按 0.1x 收费（90% 折扣）

### 5.2 Multi-turn 实战

cookbook `misc/session_memory_compaction.ipynb` 在写故事 6 turn 后量化：
- 不开 cache：每 turn 重算全部 prefix → 累积 ~10K input tokens 全价
- 开 cache：第 1 turn 创建，后续每 turn 仅"新增 1-2K"按全价 + "之前几千"按 0.1x → 总成本 **省 ~80%**

### 5.3 Background 任务的隐藏收益

session_memory_compaction.ipynb 还展示一个非常 clever 的模式：把"对话 background 摘要任务"也共享主对话的 cache：
- 摘要器和主对话共享同一段对话历史 prefix
- 摘要器只是多发一句 "summarize the above"
- → 摘要任务的 prompt 几乎全部命中 cache，单次成本降到 ~$0.003

**含义**：cache 不只是"主调用省钱"，**辅助 LLM 调用都可以蹭主对话的 cache**——这条工程价值远超官方文档强调的"重复 system prompt 省钱"。

## 6. 关键数字与 gotcha

### 6.1 硬数字

| 字段 | 5min TTL | 1h TTL |
|---|---|---|
| Cache write 价 | 1.25x base input | **2x base input** |
| Cache read 价 | **0.1x base input**（90% off） | 0.1x base input |
| 自动续期 | ✓（每次 hit） | ✓ |
| 配置 | 默认 | `"ttl": "1h"` |
| 最小可缓存（Sonnet） | 1024 tokens | 1024 tokens |
| 最小可缓存（Opus / Haiku 4.5） | 4096 tokens | 4096 tokens |
| 最大 breakpoints | 4 | 4（共用预算） |

### 6.2 致命 footgun：2026 年 3 月静默 TTL 降级事件

> 来自社区报告（DEV Community / GitHub anthropics/claude-code issues #46829, #18915）：

**事件**：2026 年 3 月初，Anthropic 在没有 changelog 公告的情况下，**把 `cache_control={"type":"ephemeral"}` 的默认 TTL 从 1 小时降到 5 分钟**。导致：
- 大量生产应用 quota 与成本飙升（cache 频繁过期 → 频繁 cache_creation）
- 客户发现是几周后

**预防（必做）**：
```python
# ❌ 不要依赖默认值
"cache_control": {"type": "ephemeral"}

# ✓ 总是显式指定
"cache_control": {"type": "ephemeral", "ttl": "5m"}  # 或 "1h"
```

这条提醒：**API 文档承诺的"默认值"在生产环境是滞后指标**——任何关键预期必须**显式写在请求里**。

### 6.3 反直觉

- **cache_creation 也算 input_tokens 计入 context window**——cache 是省钱不是省 token 位子（[[anthropic2026-context-windows]] §6.2）
- **<1024 tokens 的 prefix 静默不被 cache**：你看到 `cache_creation_input_tokens=0` 不一定是 cache 失效，可能是 prefix 太短被忽略
- **breakpoint 之后的内容不算 cached**：cache_control 在某个 block 上，意思是"这个 block 及之前都 cache，之后不"——所以**位置错了等于没开**
- **图像 cache 与文本 cache 算同一个 KV** —— 图像如果在 cache 区域内也会被 cache（但单图必须 ≥1024 token，难达到）
- **跨 region 不共享 cache**：调用不同 region endpoint 各自一份；多区域 deployment 注意账单

### 6.4 设计 footprint：哪些上层决策受 cache 影响

- **system prompt 写法**：把 dynamic 内容（用户名 / 当前时间）放 *后面*，prefix 部分纯静态——保 cache
- **tool schema**：临时加一个 tool 会让 cache 失效；**先注册全集再用**
- **多文档 RAG**：每次检索结果不同 → cache 永远 miss；要 cache 友好就**固定文档集合**或**做 retrieval 后 prompt 模板**
- **compaction 时机**：**与 cache TTL 同步**——见 §8

## 7. 限制

**Anthropic 文档承认（隐式）**：
- 5 min TTL "默认"但 2026-03 改过且没公告——**信任度下降**
- 没公开"什么时间点 cache 真的被 evict"——只承诺 TTL ≥ 而非 = 5 min
- 没说 cache 容量上限（理论上可能被 LRU 挤掉）

**我看出的**：
- **没 dashboard 看 cache 命中率** —— 客户端要自己累计 cache_read / (cache_read + input_tokens) 才能算
- **跨账户不共享**——同一个 system prompt 模板，10 万个客户每人独立 cache，整体效率低（这是产品决策不是 bug）
- **min length 1024/4096 没做"按需提示"**：开发者不知道为啥 cache 没建立；调试体验差
- **`cache_control` 的位置语义"breakpoint"** 这个抽象不直观——很多开发者第一次用都把 cache_control 放错位置

## 8. 与我关心的问题的联系（context assemble / compact / cache）

### 8.1 它是 5 件套里**最便宜**的优化原语

按性价比排序（节省 input cost / 工程复杂度）：

1. **prompt-caching**：90% off，1 行代码 → 性价比第一
2. **context-editing** (`clear_tool_uses`)：免推理，仅 mechanical
3. **token-counting**：免费但只是测量
4. **compaction**：要跑一次 LLM，最贵
5. **context-windows**：硬约束，不是工具

### 8.2 与 compaction / context-editing 的协同（critical 工程问题）

> **核心矛盾**：所有 in-session context 优化（compaction、clearing）都要**改前缀**，这意味着**让 cache 失效**。

**怎么 reconcile**：
- **compaction**：发生频率低（每次 50K+ tokens 触发），单次失效 cache 可接受；之后建立新 cache 摊销
- **context-editing**：`clear_at_least` 字段的存在就是**专门解决这个**——一次清得多让 cache 重建摊销开销
- **microcompact**（[[2604.14228-dive-into-claude-code]] §6.1）：Claude Code 实现把 cache_aware 做到极致——**等 API 真返回 `cache_deleted_input_tokens` 才下手**，不是估算

**Claude Code 源码注释（最具说服力的数据）**：
> "false path is 98% cache miss, costs ∼0.76% of fleet cache_creation"

这条注释翻译成工程语言：**不管 cache 协同，单次 compaction 让你白扔将近全部 prompt cache**。这就是为什么 cache-aware 是 critical。

### 8.3 工程化建议（managed-agent）

**Layer 1：默认全开 auto caching**
- top-level `cache_control={"type": "ephemeral", "ttl": "5m"}`，单行修改
- 监控 `cache_read_input_tokens / total_input` 命中率，<60% 是异常信号

**Layer 2：System prompt 设计 cache-aware**
- 静态前缀（角色定义 / 工具说明 / 范式约束）放最前
- 动态尾巴（user name / timestamp / per-call config）放最后
- CLAUDE.md / 工具 schema 看作"准静态"

**Layer 3：辅助 LLM 调用蹭主对话 cache**
- 像 cookbook 的 instant compaction：摘要器、auto-classifier、anomaly detector 都共享主 prefix
- 单独写新 prompt = 单独建 cache，是浪费

**Layer 4：与 compaction/editing 同步设计**
- compaction 触发后**立即重建 cache**（next call 就开 cache_control）；不要等到第二次才开
- context-editing 的 `clear_at_least` ≥ 1024（Sonnet）才有意义——清得太少 cache 失效成本 > 节省

### 8.4 不该开的场景

- **每次请求 prompt 不一样的**（如 retrieval-heavy RAG，每次检索结果不同）：cache miss 100%，付 1.25x 倍写费用纯亏
- **<1024 token 的 prefix**：被静默忽略，配置 cache_control 等于没配
- **合规 / 审计场景禁止 server-side caching**（HIPAA / GDPR 严格场合）：技术上 cache 是 server-side 临时 KV，但合规阅读上仍可能被认为是 "数据驻留"
- **极低频调用**（>1h 间隔）：5min cache 必然 miss；要么用 1h 但付 2x 写入；要么干脆别开

### 8.5 一条潜在很大但被低估的设计含义

> **Anthropic 不公开 cache 的 LRU evict 策略**——客户在高峰期可能被悄悄 evict，但 SDK 不告诉你。

**生产观测建议**：
- 每周抽样：固定 prompt 用 1h TTL 反复打，记录第 50 / 60 分钟还在不在 cache
- 出现"未到 TTL 就 miss"时降级（提前续期）

---

## 引用与相关

- 原文：<https://platform.claude.com/docs/en/build-with-claude/prompt-caching>（区域屏蔽）
- 镜像：<https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching>
- Anthropic cookbook（GitHub 可拉）：[`misc/prompt_caching.ipynb`](https://github.com/anthropics/claude-cookbooks/blob/main/misc/prompt_caching.ipynb)
- cookbook 多轮对话场景：[`misc/session_memory_compaction.ipynb`](https://github.com/anthropics/claude-cookbooks/blob/main/misc/session_memory_compaction.ipynb)
- Bedrock 文档（含相同字段）：<https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-caching.html>
- 2026-03 静默 TTL 事件：[Anthropic Silently Dropped Prompt Cache TTL from 1 Hour to 5 Minutes — DEV Community](https://dev.to/whoffagents/anthropic-silently-dropped-prompt-cache-ttl-from-1-hour-to-5-minutes-16ao)；[anthropics/claude-code#46829](https://github.com/anthropics/claude-code/issues/46829)；[anthropics/claude-code#18915](https://github.com/anthropics/claude-code/issues/18915)
- Claude Code 内部 cache 实测：[How Prompt Caching Actually Works in Claude Code](https://www.claudecodecamp.com/p/how-prompt-caching-actually-works-in-claude-code)
- 同 stack：[[anthropic2026-context-windows]] [[anthropic2026-token-counting]] [[anthropic2026-compaction]] [[anthropic2026-context-editing]]
- 综述：[[anthropic2026-context-stack]]
- 实现侧对照（5 层 compaction pipeline 中 microcompact / compaction 的 cache 协同）：[[2604.14228-dive-into-claude-code]]
- 我从哪里看到引用：用户（context engineering 研究方向）
