---
title: "Speculative Prompt Caching：把 cache_creation 延迟藏在用户打字时间里"
authors: [Anthropic]
affiliation: Anthropic
arxiv:
year: 2026
read_date: 2026-04-27
tags: [prompt-cache, context-engineering, agent-architecture]
related: [anthropic2026-context-stack, 2604.14228-dive-into-claude-code]
cited_by: []
verdict: "把 cache_creation 延迟（一次完整 prefill）藏在用户打字的 1-3 秒里——实现仅为一个 max_tokens=1 的异步 fire-and-forget 请求，但要求 prefix 字节级一致与可观测的用户思考间隙；warming 命中率低于 80% 时不划算。"
rating: 4/5
---

# Speculative Prompt Caching

## 1. TL;DR

Speculative prompt caching 是一种降低 TTFT（time-to-first-token）的工程模式：当用户开始在输入框中键入时，客户端立即向 Anthropic Messages API 异步发起一个 `max_tokens=1` 的请求，prompt 内容为带 `cache_control={"type":"ephemeral"}` 的完整 prefix（system context、文档、工具定义等）。该请求触发服务器端 prefill 并写入 KV cache，但因 `max_tokens=1` 几乎不产生 decode 开销。在用户提交真实 query 时，正式请求复用同一 prefix（须**字节级一致**），cache 命中，TTFT 仅由 query 部分的 prefill 与首 token decode 构成。在 cookbook 提供的 ~150K token SQLite 源码上下文场景中，TTFT 由 standard caching 路径下的"完整 prefill 等待"压缩到与"用户键入时间 + 单 token 解码"的并行结果。该模式的有效性条件：prefix 长度满足最小可缓存阈值（Sonnet 1024 / Opus、Haiku 4.5 4096）、用户行为存在可观测的思考间隙、warming 命中率高于 80%（cache_creation 写费用为基础 input 的 1.25 倍，需要后续命中摊销）。

## 2. 文档身份

- **底层素材**：
  - [`misc/speculative_prompt_caching.ipynb`](https://github.com/anthropics/claude-cookbooks/blob/main/misc/speculative_prompt_caching.ipynb) —— 主轴
  - [`misc/prompt_caching.ipynb`](https://github.com/anthropics/claude-cookbooks/blob/main/misc/prompt_caching.ipynb) —— 基础前置
- **来源**：Anthropic Cookbook，github.com/anthropics/claude-cookbooks
- **快照日期**：2026-04-27（platform.claude.com 区域屏蔽期间通过 cookbook 一手素材获取）

## 3. 问题背景：TTFT 的来源

### 3.1 Prefill 与 Decode 的延迟构成

Anthropic Messages API 一次推理可分两阶段：

| 阶段 | 工作 | 时间复杂度 | 可见用户感受 |
|---|---|---|---|
| Prefill | 一次性计算 prompt 中全部 token 的 K/V 矩阵并写入 GPU 内存（KV cache） | O(n) 墙钟（并行）；O(n²) 计算量 | 等待首 token 出现 |
| Decode | 自回归生成 output token，每 token 查 cache 一次 | 单 token 近似常数 | token 流式输出 |

对用户而言，TTFT 几乎完全由 prefill 决定。在 200K → 1M context window 区间，prefill 时间随长度线性增长；在 ~150K 量级的 prompt 上典型 TTFT 即可超过 10 秒。

### 3.2 Prompt cache 命中省的是 prefill

`cache_control` 字段触发的 prompt cache 复用机制本质上是**跳过 prefix 段的 prefill 计算**：服务器侧检测到当前请求的 prefix 与某条已存在的 cache 条目字节级一致，直接复用其 K/V，无需重算。在 cookbook prompt_caching.ipynb 中实测，187K token prefix 在 cache 命中后总响应时间由 ~14s 下降至 ~2.5s，下降部分主要为节省的 prefill。

### 3.3 标准 caching 在交互式应用中的弱点

标准 prompt caching 模式假设"已有 cache 可命中"。在用户首次提交某 prefix 的请求时（cold start），cache 不存在，必须先经过完整 prefill 才能写入 cache（即 `cache_creation_input_tokens` 计费段）。该次首请求的 TTFT 与无 cache 情形相同。在面向终端用户的交互场景下，这意味着用户的第一次提交会经历最长延迟，对体验最敏感的时刻反而是最慢的时刻。

## 4. Pattern：Speculative Cache Warming

### 4.1 核心观察

绝大多数交互式应用的用户存在可观测的"思考间隙"：键入 query 需要 1-5 秒、菜单选择需要数百毫秒、确认按钮点击前的犹豫等。该间隙在标准模式下被浪费——服务器空闲，用户键入。Speculative caching 利用此间隙在客户端发起后台 cache_creation 请求，使 cache 在用户提交时已处于 hot 状态。

### 4.2 触发时机

最实用的触发点：用户 focus 输入框、或用户按下第一个键。后续的键入持续时间作为 warming 的并行窗口。

### 4.3 实现要点

```python
import asyncio, copy
from anthropic import AsyncAnthropic

initial_message = {
    "role": "user",
    "content": [{
        "type": "text",
        "text": LONG_PREFIX,
        "cache_control": {"type": "ephemeral"},
    }],
}

async def sample_one_token(client, messages):
    """触发服务器端 prefill 并写入 cache，几乎不产生 decode 开销。"""
    await client.messages.create(
        model="claude-sonnet-4-6",
        messages=messages,
        max_tokens=1,
    )

# 用户 focus 输入框时立即触发
cache_task = asyncio.create_task(sample_one_token(client, [initial_message]))

# 用户键入（典型 1-5 秒）
user_question = await wait_for_user_submit()

# 等待 warming 完成（绝大多数情况下此时已 done）
await cache_task

# 正式请求；prefix 必须与 warming request 字节级一致
final_message = copy.deepcopy(initial_message)
final_message["content"].append(
    {"type": "text", "text": f"Answer the user's question: {user_question}"}
)
async with client.messages.stream(
    model="claude-sonnet-4-6",
    messages=[final_message],
    max_tokens=4096,
) as stream:
    async for text in stream.text_stream:
        ...   # 首 token 几乎瞬时到达
```

### 4.4 服务器端行为

`max_tokens=1` 请求触发完整 prefill 流程：

1. 服务器解析请求，识别 `cache_control` 标记
2. 计算 prefix 全部 token 的 K/V 矩阵
3. 写入 KV cache（`cache_creation_input_tokens` 计费）
4. 进入 decode 阶段，生成 1 个 token 后停止
5. 返回响应（output 几乎为空）

后续正式请求时，服务器在 prefill 阶段命中 cache 条目，跳过 prefix 计算，仅对追加的 query 段执行 prefill。

### 4.5 不变量：Prefix 字节级一致

Prompt cache 命中要求 prefix 部分**逐字节相同**——任何变动（包括看似无害的 timestamp 注入、参数顺序调整、空白字符）都会导致 100% miss。Cookbook 实现使用 `copy.deepcopy(initial_message)` 在追加 query 前复制整体消息结构，避免 mutation 破坏 warming request 留下的 cache 条目。

## 5. 实测数字

> 来自 `speculative_prompt_caching.ipynb` 的 SQLite 源码分析任务

### 5.1 实验设计

- **System context**：SQLite 项目的 `btree.h` + `btree.c` 源码，约 150K tokens
- **Query**："What is the purpose of the BtShared structure?"
- **模拟用户键入时间**：3 秒
- **模型**：claude-sonnet-4-6

### 5.2 两种模式对比

| 指标 | Standard Caching | Speculative Caching |
|---|---|---|
| 用户键入期间 | 服务器空闲 | 后台 warming 进行中 |
| 提交时刻 cache 状态 | cold | hot |
| 首请求 TTFT | 包含 ~150K tokens 的完整 prefill 时间 | ≈ query 部分 prefill + 单 token decode |
| 计费字段 | `cache_creation_input_tokens ≈ 150K` | warming：`cache_creation ≈ 150K`；正式：`cache_read ≈ 150K` |
| 总成本 | 1.25x × 150K | 1.25x × 150K + 0.1x × 150K + 1x × Q |

> 第二行的关键观察：speculative 模式的总 token 成本相对 standard 多了一次 `0.1x × 150K` 的 cache_read（warming 后正式请求命中），与无 speculative 的 cold start 相比并未真正节省费用——节省的是**用户感知的延迟**，而不是 token 成本。

### 5.3 cookbook 输出格式（保留 SDK 字段名）

```
Standard query statistics:
    Input tokens: <Q>
    Cache creation input tokens: ~150000
    Cache read input tokens: ---

Speculative query statistics:
    Input tokens: <Q>
    Cache creation input tokens: ---
    Cache read input tokens: ~150000
```

## 6. 实现要点（trade-off 与边界）

### 6.1 Timestamp 注入：跨 session 隔离 vs 复用

cookbook 在 prefix 中加入 `Current time: 2026-04-27 11:23:45` 字符串。该选择并非可有可无：

- **加 timestamp**：每次 session 的 prefix 独立，cache 不会被其他 session 命中（隐私 / 多租户隔离）。代价：每个 session 都要付一次 `cache_creation`
- **不加 timestamp**：prefix 在多个 session 间可复用，第二个用户的同样请求直接命中第一个用户写下的 cache。代价：cache 内容是用户共享资源，存在审计与隔离问题

实际工程中：如果 prefix 是公共资源（产品手册、API 文档），不应加 timestamp 以最大化复用；如果包含用户私有数据（其个人代码库、个人邮件），必须加 timestamp 或其他隔离标记。

### 6.2 max_tokens=1 的下限选择

Anthropic API 不接受 `max_tokens=0`（参数验证拒绝）。`max_tokens=1` 是技术下限，触发完整 prefill 但 decode 阶段在生成 1 个 token 后立即停止。该 token 内容无意义，调用方丢弃即可。

### 6.3 命中验证

每次正式请求结束后检查 `response.usage.cache_read_input_tokens` 是否接近预期 prefix 长度。若该字段为 0 或远小于 prefix，意味着 warming 失败或 prefix 不一致，需排查：

- prefix 长度是否达到最小可缓存阈值（1024 / 4096）
- warming request 与正式 request 是否字节级一致（包括 system 字段、tool schema、消息结构）
- warming 是否在 TTL 内完成（默认 5 分钟）
- model 字符串是否一致（不同模型不共享 cache）

### 6.4 异步管理

Cookbook 使用 `asyncio.create_task` 实现 fire-and-forget。生产实现需考虑：

- warming task 失败时的降级（降级为 standard caching，不阻塞用户提交）
- 用户取消输入时的 task 取消（释放连接资源；但 cache 已写入服务端，不可"回收"）
- 多次 warming 的去重（同一 prefix 的并发 warming 浪费配额）

## 7. 限制与适用场景

### 7.1 必要前提

| 条件 | 失败后果 |
|---|---|
| Prefix ≥ 1024 token (Sonnet) / 4096 token (Opus、Haiku 4.5) | `cache_control` 静默失效，warming 不写入 cache，正式请求 100% miss |
| 用户行为存在思考间隙（≥1 秒） | warming 不能完成，正式请求时 cache 未就绪 |
| Warming 与正式请求 prefix 字节级一致 | cache miss，warming 费用浪费 |
| Warming 在 cache TTL 内被命中 | cache 已 evict，正式请求重新经历 cold start |

### 7.2 经济性分析

设 prefix 长度 = N tokens，base input 单价 = p。

- 一次 warming 成本：1.25 × N × p
- 一次后续命中节省：(1 - 0.1) × N × p = 0.9 × N × p
- 命中率 h 下的净收益：h × 0.9 × N × p − 1.25 × N × p
- 净收益 ≥ 0 要求 h ≥ 1.25 / 0.9 ≈ 1.39

由于 h ≤ 1，纯成本视角下 speculative caching **永远不省钱**。其价值完全在于**用户感知延迟的下降**。该 trade-off 必须在产品决策中显式承认：用 token 成本换 TTFT，比例约为 1.25 倍 cache_creation 成本付额外延迟收益。

> 上述分析假设 warming 与正式请求是 1:1 关系。如 warming 一次 cache 后续被命中多次（同一 session 内多次提问），分子项 h × 0.9 × N × p 可线性放大，平衡点显著下移。在多轮交互场景下 speculative caching 同时具备成本与延迟优势。

### 7.3 不适用场景

- **脚本化批量调用**：无用户思考间隙可藏，warming 无法与"等待"并行
- **prefix 频繁变化**：每次请求 prefix 都不同，warming 成功也无后续命中
- **极短 prefix**（<1024 token）：达不到最小可缓存长度
- **配合 server-side compaction**（`compact_20260112`）：compaction 触发会让所有 prior cache 失效（详见 [[anthropic2026-context-stack]] §7.8），speculative warming 的产出可能在用户提交前就被 compaction 抹除
- **极低延迟容忍**：warming round-trip 本身有网络延迟（典型 100-500ms），如果用户键入时间 < warming 网络延迟，warming 在用户提交时仍未完成

### 7.4 与 1h TTL 的协同

对话型应用中用户多轮对话间隔可能超过默认 5 分钟 TTL。此场景应在 warming 与正式请求中均使用 `cache_control={"type":"ephemeral", "ttl":"1h"}`：warming 一次后整个会话期间多次命中，命中率 h 显著提升，经济性反转为正。

代价：1h TTL 的 cache write 单价为 base input 的 2 倍（vs 5min 的 1.25 倍）。需要 ≥ 2 / 0.9 ≈ 2.22 次命中才能净收益为正。

## 8. 与 stack 中其他原语的关系

### 8.1 与合订本 §6 prompt-caching 的关系

合订本（[[anthropic2026-context-stack]] §6）描述的是 cache 的 API 契约（`cache_control` 字段、TTL、定价、breakpoint 数量）与基本使用模式（auto / explicit）。本文是其延伸而非替代——讨论的是**如何在交互式系统中规避 cache cold start**这一具体工程问题，对应的是合订本未深入讨论的"用户感知延迟优化"层。

### 8.2 与 §6.8 background 任务蹭主对话 cache 的对比

| | §6.8 background sharing | 本文 speculative warming |
|---|---|---|
| 谁先到 | 主对话先创建 cache | warming 先创建 cache |
| 谁蹭谁 | 辅助 LLM 调用蹭主对话 cache | 正式请求蹭 warming cache |
| 节省的对象 | 辅助调用的 input cost | 主请求的 TTFT |
| 触发时机 | 主对话已发生后 | 用户输入开始时 |

两者方向相反、目标不同，在同一系统中可共存。

### 8.3 与 [[2604.14228-dive-into-claude-code]] 的 microcompact 对比

Claude Code 的 microcompact 与本文 speculative warming 都属于 cache-aware 设计范畴：

- **microcompact**：节流方向——等服务器返回 `cache_deleted_input_tokens` 后才决定 clear 操作的精确时机，避免清得过少导致 cache 被破坏却没释放足够 token
- **speculative warming**：预热方向——主动在用户行为间隙先种 cache 条目

两者共同点：都将 cache 状态作为一等公民纳入决策；都需要客户端有"cache 行为感知"的逻辑层。差异点：microcompact 是**反应式**（cache 已存在、决定何时清），speculative 是**先动式**（cache 不存在、决定何时种）。

### 8.4 工程化建议

在面向终端用户的 Anthropic API 应用中，speculative caching 的接入门槛低（仅一个异步任务）、风险可控（warming 失败仅退化为 standard caching）、收益直接（TTFT 下降）。建议作为以下场景的默认实践：

- 长 system prompt（≥1024 token）的对话型应用
- 文档分析、代码审查、知识库问答类工作流
- 用户输入路径包含表单填写、菜单选择等可预测的思考间隙

不建议接入的场景：自动化脚本、API 中继、prefix 高度动态的检索类应用。

---

## 引用与相关

### 一手素材
- [`misc/speculative_prompt_caching.ipynb`](https://github.com/anthropics/claude-cookbooks/blob/main/misc/speculative_prompt_caching.ipynb)
- [`misc/prompt_caching.ipynb`](https://github.com/anthropics/claude-cookbooks/blob/main/misc/prompt_caching.ipynb)

### 原文
- [Prompt caching — Claude API Docs](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)（区域屏蔽，未直接抓取）

### 相关概念脚注
- "Speculative" 一词的方法论传承自 transformer serving 优化领域的 *speculative decoding*（vLLM、Medusa、EAGLE 等），其核心思想为"用便宜的 draft model 先猜、再用主 model 验证"。本模式将"先猜后验证"的思路从 token 生成移植到 cache 状态管理：先发一个便宜请求（max_tokens=1）猜测用户会提交、再用真实请求验证（命中或重建）。

### 同 repo 关联笔记
- [[anthropic2026-context-stack]] §6（prompt caching 的 API 契约与基础用法）、§7.8（compaction 与 cache 的不兼容关系）
- [[2604.14228-dive-into-claude-code]] §6.1 microcompact（cache-aware 设计的对照实例）

### 我从哪里看到引用
用户研究方向（context assemble / compact / cache）。本文聚焦 cache 进阶模式，与合订本 §6 互补阅读。
