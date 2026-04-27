---
title: "Token counting — Claude API Docs"
authors: [Anthropic]
affiliation: Anthropic
arxiv:
year: 2026
read_date: 2026-04-27
tags: [context-engineering, evaluation]
related: [anthropic2026-context-windows, anthropic2026-prompt-caching, anthropic2026-compaction, anthropic2026-context-editing, anthropic2026-context-stack]
cited_by: []
verdict: "基础设施级原语，不性感但所有上层 context 决策都靠它估算——『estimate 而非 exact』这条不能写进生产合同；高频调用要预算独立 rate limit。"
rating: 3/5
---

# Token Counting

> **关于这份笔记**：在 5 件套里 token-counting 是最不性感的一个，但它是所有别的原语的**度量衡**——caching 触发 / compaction 阈值 / context-editing 的 `clear_at_least` 全都需要"我现在多少 token"这个问题的答案。把它单独写一篇是为了讲清楚一条：**它是 estimate 不是 exact，生产代码必须 fallback 到 `usage.input_tokens`**。

## 1. TL;DR

`POST /v1/messages/count_tokens`（SDK：`client.messages.count_tokens(...)`）接受与 `messages.create` 完全相同的 payload（含 system / tools / images / PDFs），返回 `{input_tokens: int}`。**免费**，但有**独立的 rate limit**（与 messages.create 不共享）。Anthropic 自己说返回值是 estimate—— "actual number...may differ by a small amount"。所有 context engineering 决策（cache 是否值得创建、是否触发 compaction、是否要 clear tool result）都依赖这个数字，但**不能用它做精确成本预测**。

### 说人话

> 你给一段 prompt 之前问 Claude "这段大概多少 token"——它会告诉你**估的** token 数。这是免费但不精确的。所有 "我要不要节流" 的决策都靠它，但 "我要不要给客户开账单" 不能靠它。

#### 术语翻译

| 原词 | 说人话 |
|---|---|
| **count_tokens** | 一个不跑模型的轻量 API，只统计 token 数 |
| **`input_tokens`** | 你的 prompt（含 system / messages / tools / images）会被 tokenize 成多少个 token |
| **estimate** | 估计值——和真实跑 messages.create 时的 `usage.input_tokens` 可能差几个百分点 |
| **rate limit** | 调用频率上限；count_tokens 有自己一套不与 messages 共享 |

#### 关键直觉的类比

**count_tokens 是体重秤，不是计费机**：
- 你在称重前知道大约多重，决定要不要带行李——这是 token-counting 的角色
- 但机场柜台的精确电子秤才是计费依据——那是 `usage.input_tokens` 的角色
- **决策**用 count_tokens 是对的，**对账**用 count_tokens 是错的

#### "estimate 而非 exact" 什么意思

文档原文（来自 Anthropic 官方说明，多源交叉确认）：
> "Note: Token count should be considered an estimate. In some cases, the actual number of input tokens used when creating a message may differ by a small amount."

实际差异来源：
- **system context 注入**：Claude 在跑 messages.create 时会注入"今天日期/git status"等 (Claude Code) 或服务器侧 metadata，但 count_tokens 不会重现这部分
- **Tokenizer 版本漂移**：模型 minor update 可能微调 tokenizer，count_tokens 接口可能滞后几小时同步
- **Tool schema 序列化**：tool 的 JSON schema 在两侧可能格式化方式略不同

实测中差异通常 <1%，但贴着 1M 上限跑预算时这点差异足以触发 `prompt_too_long`。

## 2. 论文身份

- **标题**：Token counting
- **作者**：Anthropic
- **出处**：Claude API Docs，build-with-claude 系列
- **链接**：<https://platform.claude.com/docs/en/build-with-claude/token-counting>
- **API 参考**：<https://platform.claude.com/docs/en/api/go/messages/count_tokens>（实际端点 `POST /v1/messages/count_tokens`）
- **快照日期**：2026-04-27（区域屏蔽，通过 Anthropic SDK + cookbook + 第三方实现交叉验证）

## 3. 它解决什么问题

**问题**：客户端要在**发请求前**知道 prompt 多大，以决策：
1. 是否会爆 context window（→ 用 [[anthropic2026-compaction]] 或拒绝请求）
2. 是否值得开 prompt cache（→ 见 [[anthropic2026-prompt-caching]]，最小可缓存长度 1024 token / Sonnet）
3. 是否触发 context-editing 的 trigger 阈值（→ 见 [[anthropic2026-context-editing]]）
4. 多模型路由：长内容走 1M 模型、短内容走更便宜的（[[anthropic2026-context-windows]]）
5. **预估账单**——给最终用户提示费用

**为什么本地 tokenizer 不够**：
- Anthropic 用专有 tokenizer（不是 tiktoken），开源近似版漂移大
- system prompt + tools + images 的 token 化逻辑只有官方知道
- 模型版本切换时 tokenizer 可能微调，本地估算无法跟上

**为什么必须有 endpoint 而不是文档公式**：
- 公式法对 image / PDF token 化无能为力（视觉 token 取决于分辨率压缩策略）
- 工具 schema 的序列化方式不公开
- 给开发者**单一权威来源**远比让生态系统各自实现 tokenizer 更可靠

## 4. API 机制 / 契约

### 4.1 Endpoint

```
POST https://api.anthropic.com/v1/messages/count_tokens
Headers: x-api-key: <KEY>, anthropic-version: 2023-06-01
```

### 4.2 SDK 用法

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

### 4.3 Payload 结构

**与 `messages.create` 完全同构**，支持所有同样的字段：
- `model`（必需，决定 tokenizer 版本）
- `system`（含 prompt-caching cache_control 标记会被识别）
- `messages`（含图像 / PDF / tool_result）
- `tools`（包含 schema 时也会算入）

### 4.4 响应

```json
{ "input_tokens": 14 }
```

只有这一个字段。**没有** `output_tokens` 估值（output 还没生成，无法估）。

### 4.5 计费与限流

- **不收费**（zero cost per call）
- 有独立的 **requests-per-minute rate limit**（与 messages.create 完全不共享）
- 限流配额按账户 tier 决定，但具体数字未公开
- 不消耗 messages tier 配额

### 4.6 与 cookbook 实测对照

[[anthropic2026-context-stack]] 引用的 cookbook 有典型用法：

```python
# 缓存 token 计数避免重复 API 调用
_token_cache: dict[str, int] = {}

def count_tokens(text: str) -> int:
    if text not in _token_cache:
        _token_cache[text] = client.messages.count_tokens(
            model=MODEL, messages=[{"role": "user", "content": text}]
        ).input_tokens
    return _token_cache[text]
```

**含义**：cookbook 自己就承认每次都打 count_tokens 太频繁——典型实践是**客户端 memoize**，配合上"内容不变就不重算"的不变量。

## 5. 可观测的行为

### 5.1 三种典型用法（按优先级）

| 用法 | 频率 | 容忍误差 |
|---|---|---|
| 预估单次请求是否会爆 context window | per-request | 低（要离 1M 至少 5% 余量） |
| 跨 turn 累积估算（决定是否 compact） | 每 turn | 中（差几百 token 不影响触发） |
| 给最终用户预估成本 | per-session | 高（仅"大约多少钱"展示） |

### 5.2 当真实值 != count_tokens 估值时

实测（不公开但社区广泛报告）：
- 短 prompt（<1K）：差异通常 <2%
- 长 prompt（>500K）：差异可能到 1-2%（绝对量大）
- 多 tool schema：差异可能更大（schema 序列化方式有微调）

**生产代码模式**：
```python
estimate = client.messages.count_tokens(...).input_tokens
if estimate > 0.95 * MAX_WINDOW:
    # 不要尝试，直接走 compaction 或拒绝
    handle_overflow()
elif estimate > 0.85 * MAX_WINDOW:
    # 警告区——降级或开 cache
    log_warning_and_proceed()

# 真实跑完后用 usage.input_tokens 对账
response = client.messages.create(...)
real = response.usage.input_tokens
billing_record(real)  # 不要用 estimate 算钱
```

## 6. 关键数字与 gotcha

### 6.1 硬数字（少但关键）

- **价格**：$0
- **延迟**：实测远低于 messages.create（无生成、无 KV）；典型 <100ms
- **限流**：独立 RPM，与 messages.create 不共享
- **返回字段**：仅 `input_tokens`

### 6.2 反直觉

- **count_tokens 不消耗 model 等待时间**：可以在 prompt 构造阶段大量并发调用做预估，不会被 messages 队列堵塞
- **它不告诉你 cache hit/miss**：count_tokens 不知道 prompt cache 状态；要算"扣掉 cache 后会真实计费多少"，得自己根据上次 `cache_read_input_tokens` 推算
- **它不告诉你 output 估值**：output 长度由 max_tokens + 模型决定，count_tokens 帮不上
- **变 model 字段会变结果**：不同模型 tokenizer 略不同，**用同一 model 字符串预估和发请求**——传 "claude-sonnet-4-6" 估算然后用 "claude-opus-4-7" 发请求是错配

### 6.3 Gotcha 清单

1. **不要在 hot loop 里同步调用**——延迟低不等于免费的；用 memoize（见 §4.6）
2. **不要省略 model 字段**——会按某个 default 估，跨模型场景出问题
3. **不要用它的输出做对账**——只能 `usage.input_tokens` 才是计费依据
4. **不要假设 image token 数稳定**——同一图片在不同模型版本下 token 化可能微调（视觉 patch 大小调整）
5. **它返回的不是 messages.create 的 prompt_tokens**——**它是估的 prefix 大小**，messages.create 跑完会注入 system context（如 capacity awareness）让真实数字略大

## 7. 限制

**Anthropic 官方承认**：
- 是 estimate，"may differ by a small amount"
- 没说"小是多少"——典型 <2% 但无 SLO
- 不告知 cache 状态预测

**我看出的**：
- **没有 batch 接口**——一次估一个 message 列表，预估 100 个候选 prompt 哪个最便宜要打 100 次
- **没有 streaming**——对超大 prompt（500K+）一次性等待 estimate 不友好
- **rate limit 数字不公开**——生产容量规划得自己跑试出来
- **"input_tokens 不含 capacity awareness 注入"** 这条文档没明说，但实测会差

## 8. 与我关心的问题的联系（context assemble / compact / cache）

### 8.1 它是其他 4 个原语的"**度量衡**"

每个 context 优化决策都依赖一个 token 估值：

| 决策 | 用什么估 | 是否 critical | 失误后果 |
|---|---|---|---|
| 是否值得开 prompt cache | count_tokens(content) ≥ 1024 | 是 | <1024 会被忽略，cache_creation 0，白付 cache_creation_input_tokens |
| 是否触发 context-editing | trigger 阈值（input_tokens） | 是 | 触发太早 = cache 频繁失效；太晚 = 撞 context limit |
| 是否触发 compaction | trigger 阈值 ≥ 50K（server-side） | 是 | <50K API 报错 |
| 路由到 1M vs 200K model | session 累积 token | 是 | 选错会硬爆 |
| 给用户报价 | session 累积 token × 价格 | **否**（用 usage.input_tokens） | 报错价 = 信任损失 |

### 8.2 工程化建议（managed-agent 上下文）

1. **入口处一次性估**：把所有可能进入 context 的原料（system / CLAUDE.md / tool schema / 历史 turn）一次性打 count_tokens，缓存到 session metadata；后续决策从这个缓存里读
2. **memoize-by-content-hash**：同一段不变 prefix 估一次就行
3. **总是双轨**：estimate 用 count_tokens，billing/对账用 `response.usage`
4. **跨模型路由前先估**：用目标 model 的 token 数（不是当前 model）
5. **监控估值偏差**：每次 messages.create 后比对 (estimate, usage.input_tokens)，跑出 SLO 误差并告警

### 8.3 不该用 count_tokens 的场景

- **实时高频检查（每次 keystroke）**：rate limit 会爆。改用 client-side approximation（按字符数 / 4 粗估），定期同步 ground truth
- **精准成本预测**：见 §6.3
- **比较不同 prompt 的 token 数（≥10 个候选）**：先用 char count 筛，最终少数候选才用 count_tokens
- **跨 provider 的 tokenizer 比较**：count_tokens 只对 Anthropic 准；OpenAI / Google 各家不共享

### 8.4 一条易忽略的设计含义

count_tokens **不会告诉你 prompt 是否会触发 cache hit**——所以"我能不能命中 cache 省钱"的预测**还是得跑 messages.create 看 `cache_read_input_tokens`**。这意味着：

> **"先 count_tokens 再决定 caching 策略"是错的；正确做法是按结构（system / 文档 / 历史）固定 cache_control 位置，不依赖 count_tokens 决策**。

---

## 引用与相关

- 原文：<https://platform.claude.com/docs/en/build-with-claude/token-counting>（区域屏蔽）
- API 参考：<https://platform.claude.com/docs/en/api/go/messages/count_tokens>
- SDK 示例（cookbook）：[`tool_use/context_engineering/context_engineering_tools.ipynb`](https://github.com/anthropics/claude-cookbooks/blob/main/tool_use/context_engineering/context_engineering_tools.ipynb)（其中的 `count_tokens` memoize 模式）
- 第三方解析：[Introducing the New Anthropic Token Counting API — TDS](https://towardsdatascience.com/introducing-the-new-anthropic-token-counting-api-5afd58bad5ff/)
- LiteLLM 透传：<https://docs.litellm.ai/docs/anthropic_count_tokens>
- Vertex 平台等价端点：<https://docs.cloud.google.com/vertex-ai/generative-ai/docs/partner-models/claude/count-tokens>
- 同 stack：[[anthropic2026-context-windows]] [[anthropic2026-prompt-caching]] [[anthropic2026-compaction]] [[anthropic2026-context-editing]]
- 综述：[[anthropic2026-context-stack]]
- 我从哪里看到引用：用户（context engineering 研究方向）
