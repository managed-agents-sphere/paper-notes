---
title: "Anthropic Context Stack：从 token 预算到跨会话记忆的五件套（合订本）"
authors: [Anthropic, Xianping (synthesis)]
affiliation: Anthropic
arxiv:
year: 2026
read_date: 2026-04-27
tags: [context-engineering, prompt-cache, memory, agent-architecture, evaluation]
related: [2604.14228-dive-into-claude-code, 2501.03276-commer]
cited_by: []
verdict: "Anthropic 把 context engineering 拆成 5 个可独立配置的 API 原语；理解它们的依赖关系（caching 是地基、count_tokens 是度量衡、clearing/compaction/memory 各管一类增长源）比掌握任何单一原语都重要——选错组合，1M context window 也救不了你。"
rating: 5/5
---

# Anthropic Context Stack：从 token 预算到跨会话记忆的五件套

> **关于这篇笔记**：这是一篇**合订本**——把 5 篇 Anthropic build-with-claude 文档的逐篇分析与跨篇综述合在一起。Part I（§4-8）逐篇拆解 5 个原语，Part II（§9-15）做综合分析。**单文件浏览，跳节阅读，整体理解**。
>
> 5 个原语：[Context windows](https://platform.claude.com/docs/en/build-with-claude/context-windows) / [Token counting](https://platform.claude.com/docs/en/build-with-claude/token-counting) / [Prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching) / [Compaction](https://platform.claude.com/docs/en/build-with-claude/compaction) / [Context editing](https://platform.claude.com/docs/en/build-with-claude/context-editing)。
>
> 因为 platform.claude.com 区域屏蔽，所有内容来自 Anthropic 官方 cookbook + SDK + 多源 WebSearch 交叉验证（2026-04-27 快照）。

## 目录

- §1 TL;DR + 说人话
- §2 文档身份
- §3 问题与动机：为什么需要 context engineering
- **Part I：五个原语**
  - §4 Context Windows — 预算合同
  - §5 Token Counting — 度量衡
  - §6 Prompt Caching — 地基
  - §7 Compaction — 重型摘要
  - §8 Context Editing — 外科手术
- **Part II：综合分析**
  - §9 五原语速览表
  - §10 依赖图
  - §11 工作流食谱（4 个场景）
  - §12 与 Claude Code 论文的对照
  - §13 取舍矩阵
  - §14 不该用的场景清单
  - §15 与我关心的问题的联系（managed-agent 工程化建议）
- §16 引用与相关

---

## 1. TL;DR + 说人话

Anthropic 把"如何在有限 context 内完成长程任务"这件事，拆成了 5 个可独立配置的 API 原语：

1. **Context windows**：你买的 token 预算（1M / 200K，按 model）
2. **Token counting**：免费的度量衡（estimate）
3. **Prompt caching**：让重复 prefix 便宜 90%（5min/1h TTL）
4. **Compaction**：触发 LLM 摘要，处理 *whole transcript* 增长（≥50K trigger，仅 4.6 系列）
5. **Context editing**：mechanical 删除 tool result（`clear_tool_uses_20250919`），处理 *sub-transcript* 增长

> **核心洞察**：5 个原语的关系是**分层依赖**——caching 是地基（让所有别的便宜）、count_tokens 是度量衡（决策依据）、clearing/compaction/memory 各管不同来源的 context 增长。**选错组合，1M context window 也救不了你**——配个昂贵 model 加全部 1M 也跑不完一个 tool-heavy 长程任务。

### 一句话每篇

| 原语 | 一句话 | 复杂度 |
|---|---|---|
| Context windows（§4） | "你的 token 预算合同；1M 是天花板不是默认" | ★ |
| Token counting（§5） | "免费的度量衡，但是 estimate 不能用于对账" | ★ |
| Prompt caching（§6） | "最便宜的优化（read 0.1x），但 prefix 一变全废" | ★★ |
| Context editing（§8） | "最被低估的原语；mechanical、lossless、sub-transcript" | ★★ |
| Compaction（§7） | "最重的（要跑 LLM）；whole-transcript、lossy、最低 50K 触发" | ★★★ |

### 说人话

> 5 件套就像**做饭**：
> - **Context windows** = 锅的大小（决定一次能煮多少菜）
> - **Token counting** = 厨房秤（先估再放）
> - **Prompt caching** = 提前洗好的常用配料（重复用快又省）
> - **Context editing** = 把吃完的盘子拿走（不动正在吃的）
> - **Compaction** = 把剩饭打包成"昨天总结"（占地方少但失原味）

### 全局术语翻译

| 原词 | 说人话 |
|---|---|
| **token** | 模型的"字"——大约 1 个英文 token = 0.75 个英文单词 = 1-3 个汉字字符 |
| **context window** | 模型一次能"看进去"的所有内容（**包括它自己的回答**），单位 token |
| **prefix** | 请求里靠前的不变部分（system prompt / 文档 / 工具定义） |
| **breakpoint** | prompt cache 块的边界标记 |
| **TTL** | Time-to-live；缓存存活时间 |
| **trigger** | 触发某个原语的阈值（typically input_tokens 或 tool_uses） |
| **whole-transcript** | 操作对象是整个对话历史（user / assistant / tool_use / tool_result 全卷入） |
| **sub-transcript** | 仅操作部分类型的 block（如只动 tool_result） |
| **lossy / lossless** | 有损 / 无损 |
| **mechanical** | 不跑 LLM，纯字符串/结构操作 |
| **applied_edits** | 服务器端响应里告诉你"刚才发生了什么 context 操作"的字段 |

---

## 2. 文档身份

- **底层素材**（5 篇 Anthropic 官方文档 URL）：
  - <https://platform.claude.com/docs/en/build-with-claude/context-windows>
  - <https://platform.claude.com/docs/en/build-with-claude/token-counting>
  - <https://platform.claude.com/docs/en/build-with-claude/prompt-caching>
  - <https://platform.claude.com/docs/en/build-with-claude/compaction>
  - <https://platform.claude.com/docs/en/build-with-claude/context-editing>
- **方法论锚点**：[Anthropic Effective Context Engineering for AI Agents (2025)](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- **实证锚点（cookbook）**：[`tool_use/context_engineering/context_engineering_tools.ipynb`](https://github.com/anthropics/claude-cookbooks/blob/main/tool_use/context_engineering/context_engineering_tools.ipynb)
- **快照日期**：2026-04-27（platform.claude.com 区域屏蔽，通过 Anthropic 官方 cookbook + SDK + 多源 WebSearch 交叉验证）

---

## 3. 问题与动机：为什么需要 context engineering

LLM 推理的根本约束是**有限的 attention 预算**。这预算的具体表现就是 context window，但问题不止"撞天花板"：

- **硬天花板**：超过 model 的 context window，API 直接 `prompt_too_long`
- **软天花板**（[context rot](https://research.trychroma.com/context-rot)）：窗口越满，模型对早期内容的召回越差——**1M 模型不是免死金牌**
- **经济天花板**：每个 turn 都要 prefill 全部 context，**线性增长 = 线性涨价 + 线性涨延迟**

Anthropic 的官方 blog 把这件事概括为：

> "context is finite with diminishing marginal returns, and the core discipline is finding the smallest set of high-signal tokens that maximize the likelihood of your desired outcome"

**"找到最小的 high-signal token 集合"** —— 这就是 context engineering 的字面定义。5 件套就是 Anthropic 给开发者的 **5 把不同尺寸的镊子**。

---

# Part I：五个原语

# §4 Context Windows — 预算合同

> **核心定位**：context-windows 不是一个可调用的 API，而是"你买了多少 context 预算"的合同。其他 4 个原语都是在花这个预算。

## 4.1 它解决什么问题

所有 transformer 模型推理时都有"一次看进去多少 token"的硬上限。这条上限是：
1. **架构性**的（attention 的二次方内存、KV cache 大小）
2. **经济性**的（更大窗口意味着更高 prefill 成本与延迟）
3. **统计性**的（context rot：窗口越满，召回越差）

**为什么仅用 caching / RAG 解决不够**：
- caching 不**消除** token，只让重复的 prefix **更便宜**
- RAG 不**消除** prompt 大小，把检索结果塞进去仍计入 window
- fine-tuning 把知识**写进权重**，但 ad-hoc 工具结果 / 用户 session 状态没法 fine-tune

## 4.2 API 契约

### 上限按 model 决定

| Model | Context window | 备注 |
|---|---|---|
| Mythos Preview | 1M | standard pricing |
| Opus 4.7 | 1M | standard pricing |
| Opus 4.6 | 1M | standard pricing |
| Sonnet 4.6 | 1M | standard pricing |
| Sonnet 4 | 1M | **tier 4 required**；>200K 按 2x input / 1.5x output 收费 |
| Sonnet 3.7 / Opus 3 / Haiku 3.5 等 | 200K | 旧型号 |

**没有显式的 API 字段** "max_context_tokens" 让你查询；这个数字是 model name 的**隐性合同**。

### 超额行为

当 input 超过 model 的 context window：
- API 返回错误 `prompt_too_long`（HTTP 400）
- **没有自动截断、没有自动摘要——调用方负责**

### Capacity awareness（agent 才有）

每次 tool call 后，Claude 收到一段 system 提示告知 remaining input token capacity。模型据此决定：
- 还能不能 read_file 一个大文件
- 是否该提前 record_finding 而不是继续 search
- 是否该 invoke memory tool 把状态外溢

> agent 不是"一上来知道窗口大小"，而是"**每一步都被告知剩余预算**"。这与 system prompt 里硬编码 `your context is 1M tokens` 是两回事——后者会让 model 误判。

## 4.3 关键数字与 gotcha

### 计费表

| Model 类 | ≤200K input | >200K 段 | output |
|---|---|---|---|
| Sonnet 4 | 1x | **2x input** | **1.5x output** |
| Opus 4.7 / 4.6 / Sonnet 4.6 / Mythos Preview | 1x | 1x | 1x |

### 反直觉

- **context window 包括 output**：max_tokens=8192 会从你的 input 预算里"预订"8192 个位子。多 turn 时累加严重
- **thinking tokens 也算**：extended thinking 模式下 thinking 输出占 window；这就是为什么有 `clear_thinking_20251015` edit type（见 §8）
- **cache_creation_input_tokens 也算 input**：第一次 cache 写入按 1.25x 计费（见 §6），但**仍然**占 context budget——cache 是省钱不是省位子
- **count_tokens 是估值**：用 §5 的 `count_tokens` 预估的数字与真实 `usage.input_tokens` 可能差几个百分点；预算贴着 1M 跑会爆

### Tier 限制

Sonnet 4 1M 需要 **tier 4 或 custom rate limit**——意味着小账户**没有这个选项**——产品差异化是真实存在的，做产品规划时不能假设客户都能用到。

## 4.4 1M 不是无副作用

文档与社区分析共同指出 4 个隐性成本（按发生顺序）：
1. **Prefill 延迟随 window 线性增长**——每个 turn 都要重新算 KV，1M 比 200K 慢 5x
2. **Cache invalidation 代价更高**——切前缀让 200K cache 失效 vs 让 1M cache 失效，重建成本不同
3. **Context rot**——召回随 window 填满下降
4. **Token 成本**：1M @ standard 也是 1M × input price，不是免费的

## 4.5 cookbook 实测

> 来自 `context_engineering_tools.ipynb` baseline 实验

任务：让 agent 读完 8 篇 ~40K token 的研究文档（共 ~330K tokens）后做 cross-doc synthesis：

| Window | 结果 |
|---|---|
| 1M | 完成；peak ~340K；早期文档细节召回明显下降（context rot） |
| 200K | 第 5 个 read_file 后 `prompt_too_long`，**任务硬失败** |

→ **1M 把 200K 的硬失败换成了 1M 的软失败（context rot）**。两种失败都需要上层原语来缓解——这就是为什么 §6/§7/§8 全是必要的。

## 4.6 限制与 gotcha 清单

- 没有公开"模型在多大 context 时性能开始下降"的曲线
- 没有 SLO 保证 capacity awareness 提示的 token 准确度
- premium pricing 的精确触发逻辑（按 prompt-level 还是 turn-level）未公开
- 文档没区分各 role（system/user/assistant/tool_result）各占多少子预算，但实际注意力权重不同
- **不要把 1M 当目标，把它当容错冗余**

---

# §5 Token Counting — 度量衡

> **核心定位**：所有 context engineering 决策（cache 是否值得创建、是否触发 compaction、是否要 clear tool result）都依赖一个 token 估值。这就是 token-counting 的角色——**5 件套里最不性感、但最基础**的一个。

## 5.1 它解决什么问题

客户端要在**发请求前**知道 prompt 多大，以决策：
1. 是否会爆 context window（→ 用 §7 compaction 或拒绝请求）
2. 是否值得开 prompt cache（→ §6，最小可缓存长度 1024 token / Sonnet）
3. 是否触发 context-editing 的 trigger 阈值（→ §8）
4. 多模型路由：长内容走 1M 模型、短内容走更便宜的（§4）
5. **预估账单**——给最终用户提示费用

**为什么本地 tokenizer 不够**：
- Anthropic 用专有 tokenizer（不是 tiktoken），开源近似版漂移大
- system prompt + tools + images 的 token 化逻辑只有官方知道
- 模型版本切换时 tokenizer 可能微调，本地估算无法跟上

## 5.2 API 契约

### Endpoint

```
POST https://api.anthropic.com/v1/messages/count_tokens
```

### SDK 用法

```python
import anthropic
client = anthropic.Anthropic()

response = client.messages.count_tokens(
    model="claude-sonnet-4-6",
    system="You are a scientist",
    messages=[{"role": "user", "content": "Hello, Claude"}],
)
print(response.input_tokens)  # → 14（举例）
```

### Payload

**与 `messages.create` 完全同构**，支持 system / messages / tools / images / PDFs。

### 响应

```json
{ "input_tokens": 14 }
```

只有这一个字段。**没有** `output_tokens` 估值（output 还没生成）。

### 计费与限流

- **不收费**（zero cost per call）
- **独立的 RPM rate limit**（与 messages.create 完全不共享）

## 5.3 关键数字与 gotcha

| 字段 | 值 |
|---|---|
| 价格 | $0 |
| 延迟 | 通常 <100ms（无生成、无 KV） |
| 限流 | 独立 RPM，与 messages.create 不共享 |
| 返回字段 | 仅 `input_tokens` |

### "estimate 而非 exact"

文档原文：
> "Note: Token count should be considered an estimate. In some cases, the actual number of input tokens used when creating a message may differ by a small amount."

**实际差异来源**：
- system context 注入（capacity awareness 等）只有 messages.create 才会真发生
- tokenizer 版本漂移
- tool schema 序列化方式差异

实测中差异通常 <1-2%，但贴着 1M 上限跑预算时这点差异足以触发 `prompt_too_long`。

### 反直觉

- **count_tokens 不告诉你 cache hit/miss**——要算"扣掉 cache 后会真实计费多少"得自己根据上次 `cache_read_input_tokens` 推算
- **它不告诉你 output 估值**——output 长度由 max_tokens + 模型决定
- **变 model 字段会变结果**：不同模型 tokenizer 略不同，**用同一 model 字符串预估和发请求**

## 5.4 cookbook 实测模式（必抄）

```python
# 客户端 memoize 避免重复 API 调用
_token_cache: dict[str, int] = {}

def count_tokens(text: str) -> int:
    if text not in _token_cache:
        _token_cache[text] = client.messages.count_tokens(
            model=MODEL, messages=[{"role": "user", "content": text}]
        ).input_tokens
    return _token_cache[text]
```

**含义**：cookbook 自己就承认每次都打 count_tokens 太频繁——典型实践是**客户端 memoize**。

## 5.5 生产代码模式

```python
estimate = client.messages.count_tokens(...).input_tokens
if estimate > 0.95 * MAX_WINDOW:
    handle_overflow()           # 直接走 compaction 或拒绝
elif estimate > 0.85 * MAX_WINDOW:
    log_warning_and_proceed()   # 警告区——降级或开 cache

# 真实跑完后用 usage.input_tokens 对账
response = client.messages.create(...)
real = response.usage.input_tokens
billing_record(real)            # 不要用 estimate 算钱
```

## 5.6 限制与 gotcha 清单

- **是 estimate**，不能用于精确计费预测
- **没有 batch 接口**——一次估一个，预估 100 个候选要打 100 次
- **没有 streaming**——超大 prompt 一次性等待 estimate 不友好
- **rate limit 数字不公开**——生产容量规划得自己跑试出来
- 不要在 hot loop 里同步调用——延迟低不等于免费
- 不要省略 model 字段——会按某个 default 估，跨模型场景出问题
- 不要用它的输出做对账——只能 `usage.input_tokens` 才是计费依据

---

# §6 Prompt Caching — 地基

> **核心定位**：5 件套里**最便宜的优化原语**（read 0.1x，省 90%），也是其他原语必须协同的**地基**——所有改前缀的操作（compaction、context-editing）都会让 cache 失效。

## 6.1 它解决什么问题

Transformer 推理时 prefix 的 KV computation 是 attention 大头开销。同一段长 prefix 在多次调用里被重新算 N 次，是浪费。

**为什么 client-side caching 不够**：
- 你能 cache 自己的字符串，但**算 attention KV 的是服务器**
- API gateway 没法 cache 模型推理中间状态

## 6.2 API 契约

### Auto caching（推荐）

```python
response = client.messages.create(
    model="claude-sonnet-4-6",
    cache_control={"type": "ephemeral"},   # ← top-level
    messages=[{"role": "user", "content": LONG_PREFIX + user_question}],
)
```

SDK 自动把 breakpoint 放在**最后一个可缓存 block**。多 turn 时**自动前移**。

### Explicit breakpoints

```python
messages=[{
    "role": "user",
    "content": [
        {"type": "text", "text": SYSTEM_DOCS,
         "cache_control": {"type": "ephemeral"}},
        {"type": "text", "text": user_question}
    ]
}]
```

**最多 4 个 breakpoints**（automatic caching 用掉 1 个 slot）。每个独立 TTL。

### 1h TTL（extended）

```python
"cache_control": {"type": "ephemeral", "ttl": "1h"}
```

- write 价 **2x base input**（vs 5min 的 1.25x）
- read 价相同 **0.1x base input**

### Multi-turn 自动前移（关键模式）

cookbook `misc/prompt_caching.ipynb` 实测的 multi-turn 表：

| Request | Cache 行为 |
|---|---|
| Turn 1 | System + User:A 写入 cache |
| Turn 2 | System + User:A 命中；Asst:B + User:C 写入 |
| Turn 3 | System...User:C 命中；Asst:D + User:E 写入 |

**含义**：第二轮起，几乎 100% input tokens 都从 cache 来；只有"最新一句话 + 上一句的回答"按全价。

### Usage 字段

```python
usage.input_tokens                  # 真实新输入 token
usage.cache_creation_input_tokens   # 新写入 cache 的 token（按 1.25x 或 2x）
usage.cache_read_input_tokens       # 命中的 cache token（按 0.1x）
usage.cache_creation                # 子对象（开了 1h TTL 时存在）
usage.cache_creation.ephemeral_5m_input_tokens
usage.cache_creation.ephemeral_1h_input_tokens
```

## 6.3 关键数字

| 字段 | 5min TTL | 1h TTL |
|---|---|---|
| Cache write 价 | 1.25x base input | **2x base input** |
| Cache read 价 | **0.1x base input**（90% off） | 0.1x base input |
| 自动续期 | ✓（每次 hit） | ✓ |
| 配置 | 默认（**但见 §6.5**） | `"ttl": "1h"` |
| 最小可缓存（Sonnet） | 1024 tokens | 1024 tokens |
| 最小可缓存（Opus / Haiku 4.5） | 4096 tokens | 4096 tokens |
| 最大 breakpoints | 4 | 4（共用预算） |

## 6.4 cookbook 实测：单轮加 cache

cookbook 跑 *Pride and Prejudice*（~187K tokens）：

| 场景 | 时间 | input_tokens | cache_create | cache_read |
|---|---|---|---|---|
| 无 cache | ~14s | 187,000 | 0 | 0 |
| 第一次开 cache（写入） | ~14s | ~50 | ~187,000 | 0 |
| 第二次同 prompt（命中） | ~2.5s | ~50 | 0 | ~187,000 |

**关键发现**：第一次延迟与无 cache 相近（要先算 KV 再存）；第二次延迟降 **>5x**；第二次按 0.1x 收费（90% 折扣）。

## 6.5 致命 footgun：2026 年 3 月静默 TTL 降级事件

> 来自社区报告：[DEV Community](https://dev.to/whoffagents/anthropic-silently-dropped-prompt-cache-ttl-from-1-hour-to-5-minutes-16ao) / [anthropics/claude-code#46829](https://github.com/anthropics/claude-code/issues/46829) / [#18915](https://github.com/anthropics/claude-code/issues/18915)

**事件**：2026 年 3 月初，Anthropic 在没有 changelog 公告的情况下，**把 `cache_control={"type":"ephemeral"}` 的默认 TTL 从 1 小时降到 5 分钟**。导致大量生产应用 quota 与成本飙升（cache 频繁过期 → 频繁 cache_creation）。

**预防（必做）**：
```python
# ❌ 不要依赖默认值
"cache_control": {"type": "ephemeral"}

# ✓ 总是显式指定
"cache_control": {"type": "ephemeral", "ttl": "5m"}
```

**教训**：API 文档承诺的"默认值"在生产环境是滞后指标——**任何关键预期必须显式写在请求里**。

## 6.6 cache 命中条件（严格）

要命中前缀必须**逐字节相同**。常见命中失败：
- prepend 一个时间戳到 user message → 永远 miss（cookbook 用这个手段做 baseline 对照）
- system prompt 改了一句 → 整个 cache 重建
- tool schema 变化 → 整个 cache 重建
- 模型 minor version 切换（如 sonnet-4-6 → 4-7）→ 不共享 cache

## 6.7 反直觉

- **cache_creation 也算 input_tokens 计入 context window**——cache 是省钱不是省 token 位子
- **<1024 tokens 的 prefix 静默不被 cache**：你看到 `cache_creation_input_tokens=0` 不一定是 cache 失效，可能是 prefix 太短被忽略
- **breakpoint 之后的内容不算 cached**：cache_control 在某个 block 上意味"这个 block 及之前都 cache，之后不"——位置错了等于没开
- **跨 region 不共享 cache**：调用不同 region endpoint 各自一份；多区域 deployment 注意账单

## 6.8 Background 任务的隐藏收益（cookbook 黄金模式）

`misc/session_memory_compaction.ipynb` 展示一个非常 clever 的模式：把"对话 background 摘要任务"也共享主对话的 cache：
- 摘要器和主对话共享同一段对话历史 prefix
- 摘要器只是多发一句 "summarize the above"
- → 摘要任务的 prompt 几乎全部命中 cache，单次成本降到 ~$0.003

**含义**：cache 不只是"主调用省钱"，**辅助 LLM 调用都可以蹭主对话的 cache**。

## 6.9 限制与 gotcha 清单

- 5 min TTL "默认"但 2026-03 改过且没公告——**信任度下降，必须显式 ttl**
- 没公开"什么时间点 cache 真的被 evict"——只承诺 TTL ≥ 而非 = 5 min
- 没说 cache 容量上限（理论上可能被 LRU 挤掉）
- 没 dashboard 看 cache 命中率——客户端要自己累计
- 跨账户不共享——同一个 system prompt 模板，10 万个客户每人独立 cache

---

# §7 Compaction — 重型摘要

> **核心定位**：5 件套里**最重的**原语——唯一会跑一次 LLM 推理。最 lossy 也最强（处理 *whole transcript*，不只是 tool result）。新 server-side `compact_20260112` 仅 Opus 4.6 / Sonnet 4.6 支持。

## 7.1 它解决什么问题

长程 agent / 多 turn chatbot 的 context 单调增长，最终撞 §4 上限——或*在撞上限前*已经因 *context rot* 召回下降。

**为什么 caching / clearing 单独不够**：
- §6 让重读便宜，**不删** token——预算还是会爆
- §8 只能删 tool result，**user / assistant 文字不动**——长对话里很多增长来自模型自己的回答和用户提问，clearing 管不了
- 必须有一个**全 transcript 的 lossy 压缩**才能管所有增长源

## 7.2 API 契约：两条实现路径

### Server-side（推荐 Opus 4.6+）

```python
response = client.beta.messages.create(
    model="claude-opus-4-6",
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

### SDK-side（旧 model / 灵活摘要）

```python
runner = client.beta.messages.tool_runner(
    model="claude-sonnet-4",
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
- 仅 `tool_runner` 抽象支持
- 仅 SDK >= 0.74.1
- 不与 server-side web search / extended thinking 兼容（cache 累积导致过早触发）

## 7.3 输出格式

server-side 触发后，response 里会插入一个 typed content block：

```python
{"type": "compaction", "content": [{"type":"text", "text":"<summary>...</summary>"}]}
```

下一次请求里**把这个 block 原样塞回 messages**，服务端会自动**丢弃此 block 之前的所有 messages**：

```python
next_request_messages = [
    {"role": "assistant", "content": [
        {"type": "compaction", "content": last_compaction_block.content}
    ]},
    {"role": "user", "content": "继续"}
]
```

## 7.4 默认 summarization prompt（公开）

cookbook 引用的官方默认 prompt 全文（值得记住）：

> You have written a partial transcript for the initial task above. Please write a summary of the transcript. The purpose of this summary is to provide continuity so you can continue to make progress towards solving the task in a future context, where the raw history above may not be accessible and will be replaced with this summary. Write down anything that would be helpful, including the state, next steps, learnings etc. You must wrap your summary in a `<summary></summary>` block.

## 7.5 `instructions` 是**完全替换**

```python
"edits": [{
    "type": "compact_20260112",
    "trigger": {...},
    "instructions": "Focus on preserving code snippets, variable names, and technical decisions."
}]
```

**重要陷阱**：你提供 `instructions` 时，默认 prompt 整段被替换——你必须**自己负责完整 framing**（包括"包在 `<summary>` 标签里"这种格式约束）。一句简短 hint 是不够的；**最佳实践是附加到默认 prompt 之上写**：

```python
"instructions": """[default prompt 全文 here]

Additionally, focus on preserving:
- Specific lifespan numbers per organism
- The exact text of the user's correction in turn 5"""
```

## 7.6 cookbook 实测

### 客户服务工单工作流（SDK-side）

任务：5 张工单，每张 7 个 tool call。

| 配置 | 总 turns | 累积 input | 触发次数 |
|---|---|---|---|
| 无 compaction | 27 | ~150,000 | 0 |
| `compaction_control={enabled:True, threshold:5000}` | ~30 | ~79,000 | 2 |

**节省 ~47% input tokens**。

### 研究 agent（server-side）

| 配置 | Peak context | 是否完成 |
|---|---|---|
| Baseline（1M window，无 management） | ~340K | ✓（但 context rot） |
| Baseline（200K window） | 200K | ✗（hit limit） |
| `compact_20260112` trigger=180K | ~210K（compaction 后回落） | ✓ |

### high-level vs obscure 保留率

cookbook 实测 probe：
- ✓ 高层级事实保留：3/3
- ✗ 偶发细节保留：0/3 - 1/3

→ 默认 prompt 让模型按"任务连续性"取舍——会保留你**任务上需要的**，丢弃**任务上不需要的**。

## 7.7 关键数字

| 字段 | Server-side | SDK-side |
|---|---|---|
| 最低 trigger | **50,000 input_tokens** | 无下限（太低会自循环） |
| 默认 trigger | server-adaptive | 100,000 |
| 模型支持 | Opus 4.6 / Sonnet 4.6 | 大多数（含 4.5 / 4） |
| Beta header | `compact-2026-01-12` | 无 |
| 推理成本 | 1 次 LLM call（用主 model） | 1 次（可指定更便宜 model） |

## 7.8 反直觉与 Gotcha

- **compaction 不能频繁触发**：每次 = 一次 LLM 推理 + cache 失效
- **compaction summary 本身也消耗 token**：~5-10K 是典型摘要大小
- **不能用 server-side compaction 配合 server-side web search / extended thinking**（cookbook 明示）
- **prior compaction blocks 会被卷入下次摘要**：长会话连续 compaction 会层层稀释最早的信息
- **`<summary></summary>` 标签是软约束，模型偶尔不遵守**（cookbook 注释 "tags aren't parsed"）
- **不要假设 instructions 会追加到默认 prompt**——它**替换**全部
- **不要对 compaction 后的对话期望 cache hit**：摘要替换全部历史 = prefix 全变 = cache 全失效
- **不要用 compaction 持久化跨 session**：摘要是 session 内的 typed block

## 7.9 限制清单

- "lossy by design"——specific numbers / verbatim phrasing 会丢
- 仅 Opus 4.6 / Sonnet 4.6（server-side）
- 没有 dry-run 模式
- 没有 callback hook（[[2604.14228-dive-into-claude-code]] 的 PreCompact hook 在 API 端不存在）

---

# §8 Context Editing — 外科手术

> **核心定位**：5 件套里**最被低估的原语**——免推理、lossless（如工具可重调）、surgical（仅清 tool_result，不动 user/assistant）。`clear_at_least` 字段是 cache-aware 设计的活化石。

## 8.1 它解决什么问题

长程 agent 的 context 增长**主要来自工具结果**，不是对话本身。每次 read_file 30-100K token、每次 search 几 K token 都会累积，且**之后每个 turn 都要传**——线性增长很快撞 §4 上限。

**为什么 §7 不是最优解**：compaction 跑一次 LLM——贵；compaction lossy——文档的具体细节会被压成"读过该文件"。

**为什么 §6 不是最优解**：caching 让重读 cheap，**不删** token——预算还是会爆。

## 8.2 API 契约

### 完整字段示例

```python
from anthropic.types.beta.beta_context_management_config_param import (
    BetaContextManagementConfigParam,
)

config: BetaContextManagementConfigParam = {
    "edits": [{
        "type": "clear_tool_uses_20250919",
        "trigger": {"type": "input_tokens", "value": 30_000},
        "keep": {"type": "tool_uses", "value": 3},
        "clear_at_least": {"type": "input_tokens", "value": 5_000},
        "exclude_tools": ["web_search"],
    }]
}

response = client.beta.messages.create(
    betas=["context-management-2025-06-27"],
    context_management=config,
    messages=[...],
    tools=[...],
)
```

### 字段语义

| 字段 | 类型 | 含义 |
|---|---|---|
| `type` | `clear_tool_uses_20250919` 或 `clear_thinking_20251015` | edit 类型 |
| `trigger` | `{"type":"input_tokens"\|"tool_uses", "value":N}` | 多大触发 |
| `keep` | `{"type":"tool_uses", "value":N}` | 保留最近 N 个 tool_result |
| `clear_at_least` | `{"type":"input_tokens", "value":N}` | **至少**释放 N tokens 才清 |
| `exclude_tools` | `string[]` | 不清这些 tool 的 result（按 tool name） |

**`trigger` vs `clear_at_least` 的区别**：
- `trigger`：决定"什么时候考虑清理"
- `clear_at_least`：决定"决定清理时至少清多少"——保护一次清得有意义

### 双 edit 协同（thinking + tool_result）

```python
"edits": [
    {"type": "clear_thinking_20251015", "trigger": {...}},   # 必须先
    {"type": "clear_tool_uses_20250919", "trigger": {...}},  # 再
]
```

> **关键**：`clear_thinking_20251015` 必须放 edits 数组**首位**（cookbook 明示）

### 响应字段（observable，唯一）

每次 messages.create 返回，如果触发了 clearing：

```python
response.context_management.applied_edits
# 形如：
# [{"edit_type": "clear_tool_uses_20250919",
#   "tool_uses_cleared": 7,
#   "tokens_cleared": 42_000}]
```

> **5 件套里唯一明确告诉你"刚才发生了什么 context 操作"的字段**。

### 与 memory tool 的关键协同

```python
"edits": [{
    "type": "clear_tool_uses_20250919",
    "trigger": {...},
    "exclude_tools": ["memory"],   # ← critical
}]
```

不写 `exclude_tools: ["memory"]` 的后果：
- agent 调 memory tool 写入"用户偏好"
- clearing 触发，把 memory tool result 也清了
- agent 再读 memory 时**找不到自己刚写的**——无限重写循环

## 8.3 cookbook 实测

> 来自 `context_engineering_tools.ipynb`

### Research agent baseline vs clearing

任务：8 篇 ~40K token 文档读 + synthesis。

| 配置 | Peak context | 完成？ | 触发次数 |
|---|---|---|---|
| Baseline（无 management） | ~340K | ✓（context rot） | - |
| `trigger=30K, keep=4, clear_at_least=10K` | <60K | ✓ | 多次 |

**关键观察**：每次 clearing 让 context "step-down"——不是平滑降，是阶梯式：清完掉一截、继续读、再爬升、再清。多次清理的曲线呈锯齿形。

### File reads count

```
File reads: baseline=8, clearing=11
```

**含义**：clearing 触发后，agent 偶尔需要 *重读* 早些清掉的文件——但代价远小于 compaction 跑摘要。

### 三原语 all-three 联用 timeline

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

**clearing 的"step"小但频繁；compaction 的"drop"大但稀少**——这是 cookbook 的核心 visualization。

## 8.4 关键数字

| 字段 | 默认 / 限制 |
|---|---|
| Beta header | `context-management-2025-06-27` |
| `clear_thinking_20251015` 位置 | 必须 edits 数组首位（如启用） |
| 推理成本 | **0**（mechanical edit） |
| 触发判断时机 | server-side, prefill 后 |
| `exclude_tools` 配 memory 的必要性 | **必须**（否则 memory 自我清空） |
| `applied_edits` 可观测 | ✓（5 件套唯一） |

## 8.5 `clear_at_least` 的数学含义（cache-aware 精髓）

设：
- 当前 cached prefix = C tokens
- 一次 cache_creation 成本 ≈ 1.25 × C × base_input_price
- 一次 cache_read 成本 ≈ 0.1 × C × base_input_price

**为了让 clearing 净收益 > 0**：

```
next_call_tokens_saved >  1.15 × C
（等价：clear_at_least > C × 1.15）
```

意思：**清的量必须超过现有 cache 的 1.15 倍才划算**。所以：
- 如果你 cache 30K，`clear_at_least` 应该 ≥ 35K（不是 5K！）
- 否则每次清都让 cache 重建，反而亏

cookbook 默认 `clear_at_least=5K` 是因为 cache prefix 还小；生产中要按当前 cache 大小动态调。

## 8.6 反直觉与 Gotcha

- **clearing 不省 latency 多少**：tool_use 块还在，model 仍要 attention 它们——主要省的是钱（input cost）和推迟撞 context limit
- **`tool_use` block 保留是关键设计**：模型记得它做过什么，只是看不到结果——这让"重读 same file" 变成模型的自然选择
- **clearing 改前缀让 prompt cache 失效**：和 compaction 一样——`clear_at_least` 就是为此存在
- **不能选择性清"某 tool 的某次结果"**：粒度是"所有 tool 的最后 keep 个之外"
- **`keep` 是按 tool 数不是按 token 数**：keep=5 意味着保留最新 5 个 tool result——不管它们多大
- **trigger 类型可以是 tool_uses**：某些 tool-heavy 工作流这比 token 计数更稳定
- 不要漏 `exclude_tools: ["memory"]`
- 不要用 keep=1：留太少导致 agent 立刻 re-fetch
- 不要把 clear_thinking 放第二位
- 不要假设 lossless 适用所有 tool：rate-limited / 计费 API 重调有真实代价
- 不要遗忘 betas header

## 8.7 限制清单

- 仅清 tool_result 和 thinking，**user / assistant 文字增长 clearing 帮不上**——必须靠 §7
- 没有"预演"机制：和 compaction 一样，`clear_at_least` 设错也得真触发后才知道
- 没有 callback hook：清完以后想做点什么没有 hook
- 粒度限制：不能"清掉某 tool 名下的 result"——要么 exclude 全部，要么不动
- thinking edit type 还在 beta，可能会改

---

# Part II：综合分析

---

## 9. 五原语速览表

| 原语 | 操作对象 | 性质 | 关键触发字段 | 主要成本 | 解决的问题 |
|---|---|---|---|---|---|
| **Context-windows**（§4） | 总预算 | model-bound | （隐性）选 model | 算 token 单价 | 元约束 |
| **Token-counting**（§5） | prompt 估值 | mechanical | 调用 `count_tokens` | 0（免费） | 决策辅助 |
| **Prompt-caching**（§6） | prefix KV | server-side cache | `cache_control={ttl:5m\|1h}` | write 1.25x/2x；read 0.1x | 重复 prefix 太贵 |
| **Context-editing**（§8） | sub-transcript（tool_result + thinking） | mechanical | `trigger`/`keep`/`clear_at_least`/`exclude_tools` | 0（无推理）+ 工具重调 | tool-heavy 累积 |
| **Compaction**（§7） | whole transcript | 1 LLM 推理 | `trigger.value ≥ 50K` + 可选 `instructions` | 1 次 LLM call + cache 全失效 | 对话+模型推理累积 |

> 还有第 6 个相关原语：**memory tool**（`memory_20250818`）——它跨 session，不属于 build-with-claude 这 5 篇 docs 的范畴，但是 cookbook 里和这 5 件套强协同（见 §11、§14、§15）。

---

## 10. 依赖图

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

1. **Prompt-caching 是地基** —— 所有别的优化都建在 cache 之上，因为别的原语**改 prefix = cache 失效**
2. **Token-counting 是度量衡** —— 没它，trigger 阈值是瞎设
3. **Clearing 和 compaction 是孪生敌友** —— 它们都改 prefix（破 cache），但操作对象互补（sub-transcript vs whole-transcript），所以可以**联用**而不是取舍

---

## 11. 工作流食谱（4 个场景）

按"任务复杂度 × context 增长源"分场景：

### 11.1 单轮长 context QA（"读这篇 200K 论文回答问题"）

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

### 11.2 多 turn chatbot（"几十轮对话，无 tool"）

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

### 11.3 长程 agent（"连续读 50 个文件、调 100 次 tool"）

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

### 11.4 跨会话研究 agent（"分多次会话写一篇论文"）

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
- compaction trigger 略低（180K），先于 clearing；compaction 处理*对话*累积、clearing 处理*tool*累积，互不抵消
- memory tool 跨 session 持久化关键发现
- 1h TTL 保护跨长 session 间隔

---

## 12. 与 Claude Code 论文（[[2604.14228-dive-into-claude-code]]）的对照

Claude Code 论文笔记描述了 Claude Code v2.1.88 实现的 **5 层 compaction pipeline**。把这 5 层和 5 件套 API 原语对照：

| Claude Code 层 | 等价 API 原语 | 备注 |
|---|---|---|
| Budget reduction（per-tool-result 限长） | （API 无） | 客户端 only：tool 结果超过 maxResultSizeChars 替换为 reference |
| Snip（older history trim） | （API 无） | 客户端 only：lightweight FIFO trim |
| Microcompact（cache-aware） | ≈ §8 `clear_tool_uses_20250919` | Claude Code 等 API 真返回 `cache_deleted_input_tokens` 才下手——比 API 默认的估算更精准 |
| Context collapse（read-time projection） | （**API 无**） | Claude Code 独家发明：原 history 不动，只改"读出来时呈现成什么样" |
| Auto-compact（LLM summary） | ≈ §7 `compact_20260112` | Claude Code 用 PreCompact hook 让 user 注入指令；API 端用 `instructions` |

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

---

## 13. 取舍矩阵

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

---

## 14. 不该用的场景清单

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

---

## 15. 与我关心的问题的联系（managed-agent 工程化建议）

这一段是**最重要的**——读完前 14 节后，把它对应到我们的 managed-agent 实践：

### 15.1 默认配置模板（直接抄）

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

### 15.2 监控 dashboard（生产必接）

至少接 4 个观测点：

1. **`cache_read_input_tokens / total_input` 命中率**：<60% 是异常信号
2. **`applied_edits.tokens_cleared` 累计**：每 session 多少 token 被 clearing 释放
3. **`compaction` block 出现次数**：高频意味着 trigger 设低、要调
4. **`usage.input_tokens vs count_tokens estimate` 偏差**：>2% 是 tokenizer 漂移信号

### 15.3 与 Claude Code 抄作业要点

[[2604.14228-dive-into-claude-code]] §8.2 列了 3 件 Claude Code 做对了：
- 渐进式（便宜的先）
- cache-aware（等真返回 cache_deleted 才下手）
- read-time projection（不动原 history）

5 件套 API 给了 1、2 但没给 3。我们做 managed-agent 时**第 3 条要自建**：
- compaction summary 与原 history 必须并存
- 可以做"compaction view" 隐藏在 UI 后，原 transcript 落盘
- resume / fork / rewind 必须能拿回原始数据

### 15.4 "选模型 = 选 context 策略"

这是 5 件套综合后最反直觉但最重要的结论：

| 模型 | 1M | server-side compaction | 默认应用策略 |
|---|---|---|---|
| Opus 4.7 | ✓ | ✗（仅 4.6） | caching + clearing |
| Opus 4.6 | ✓ | ✓ | 全 5 件套 |
| Sonnet 4.6 | ✓ | ✓ | 全 5 件套 |
| Sonnet 4 | ✓（tier 4，>200K 2x 价） | ✗ | caching + clearing；超 200K 路由到 4.6 |
| 更老 | 200K | ✗ | caching + SDK-side compaction |

**模型选型 ≠ 只看能力 / 价格**，要把"context 策略可用性"加进矩阵。

### 15.5 与 ComMer（[[2501.03276-commer]]）的对照（latent vs textual compaction）

ComMer 是 **latent**（连续向量）压缩：风格类记忆压得好、事实记忆 destructive interference。Compaction 是 **textual**（自然语言摘要）：
- ✓ 没有 destructive interference 问题（事实可保留）
- ✗ 不能压到很小（latent 可压 32 token；textual 摘要至少千字）
- ✓ 模型直接读懂（latent 需要训练）
- ✗ 有损（重新表述会引入误差）

→ textual compaction 是 *online* 操作（无需训练）、对所有 model 可用——这是 API 选 textual 路线的根本原因。

**结论**：managed-agent 的 memory 系统**不是二选一**：
- 用户偏好 / 口吻 → ComMer 路线（latent）
- 工具结果 / 错误日志 / 可检索事实 → §8 clearing + memory tool
- 长对话历史 → §7 compaction

### 15.6 前瞻：还缺什么

5 件套 + memory tool 解决了大部分 in-session / cross-session 场景，但仍有缺口：

- **可逆性**：API 没 read-time projection；想做 audit 必须客户端落盘双轨
- **Hook**：compaction 没 PreCompact callback；要做"摘要前先存关键 fact"必须客户端代码做
- **细粒度**：context-editing 不能选择性删某 tool 的某次结果；要么 exclude 全部要么不动
- **跨账户共享 cache**：每个客户独立 cache，整体效率有上限
- **observable 不一致**：5 个原语的 telemetry 字段命名不统一，客户端要做适配层

这些是下一代 API 可能演进的方向，也是**managed-agent 平台需要自己补的**抽象层。

---

## 16. 引用与相关

### 5 个原语原文（platform.claude.com 区域屏蔽）

- <https://platform.claude.com/docs/en/build-with-claude/context-windows>
- <https://platform.claude.com/docs/en/build-with-claude/token-counting>
- <https://platform.claude.com/docs/en/build-with-claude/prompt-caching>
- <https://platform.claude.com/docs/en/build-with-claude/compaction>
- <https://platform.claude.com/docs/en/build-with-claude/context-editing>

### Anthropic 官方 cookbook（GitHub 可拉，本笔记主要素材源）

- [`tool_use/context_engineering/context_engineering_tools.ipynb`](https://github.com/anthropics/claude-cookbooks/blob/main/tool_use/context_engineering/context_engineering_tools.ipynb) —— **三原语对比金矿**（compaction vs clearing vs memory）
- [`tool_use/automatic-context-compaction.ipynb`](https://github.com/anthropics/claude-cookbooks/blob/main/tool_use/automatic-context-compaction.ipynb) —— SDK 端 compaction
- [`misc/session_memory_compaction.ipynb`](https://github.com/anthropics/claude-cookbooks/blob/main/misc/session_memory_compaction.ipynb) —— 客户端 instant compaction 模式
- [`misc/prompt_caching.ipynb`](https://github.com/anthropics/claude-cookbooks/blob/main/misc/prompt_caching.ipynb) —— caching 完整 demo

### SDK 字段权威

- [`anthropic-sdk-python/examples/memory/basic.py`](https://github.com/anthropics/anthropic-sdk-python/blob/main/examples/memory/basic.py) —— `BetaContextManagementConfigParam` 完整字段集

### Anthropic blog & launch

- [Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [Managing context on the Claude Developer Platform](https://claude.com/blog/context-management)
- [Introducing Claude Opus 4.6](https://www.anthropic.com/news/claude-opus-4-6)
- [Claude Sonnet 4 now supports 1M tokens of context](https://www.anthropic.com/news/1m-context)

### 第三方深度解析

- [Anthropic Silently Dropped Prompt Cache TTL from 1 Hour to 5 Minutes — DEV Community](https://dev.to/whoffagents/anthropic-silently-dropped-prompt-cache-ttl-from-1-hour-to-5-minutes-16ao)
- [anthropics/claude-code#46829](https://github.com/anthropics/claude-code/issues/46829)、[#18915](https://github.com/anthropics/claude-code/issues/18915) —— TTL 静默事件 GitHub issues
- [Context Editing Looks Like a Feature. It's Actually a Garbage Collector Without Write Barriers](https://conikeec.substack.com/p/context-editing-looks-like-a-feature)
- [Context Compaction API Complete Guide — Claude Lab](https://claudelab.net/en/articles/api-sdk/compaction-api-context-management)
- [How Prompt Caching Actually Works in Claude Code](https://www.claudecodecamp.com/p/how-prompt-caching-actually-works-in-claude-code)

### 平台等价端点（参考实现）

- Bedrock：<https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-caching.html>、<https://docs.aws.amazon.com/bedrock/latest/userguide/claude-messages-compaction.html>
- Vertex AI：<https://docs.cloud.google.com/vertex-ai/generative-ai/docs/partner-models/claude/count-tokens>

### Memory tool（强协同对象）

- <https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool>

### 同 repo 相关笔记

- [[2604.14228-dive-into-claude-code]] —— **实现侧对照**（5 层 compaction pipeline 与 5 件套 API 的映射，§12 详述）
- [[2501.03276-commer]] —— **latent vs textual compaction 边界**（§15.5 详述）

### Context rot 实证

- <https://research.trychroma.com/context-rot>

### 我从哪里看到引用

用户研究方向（context assemble / compact / cache）。这篇合订本是与 Claude Code paper 配套阅读的"API 侧"教科书。
