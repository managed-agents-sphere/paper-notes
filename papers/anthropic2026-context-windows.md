---
title: "Context windows — Claude API Docs"
authors: [Anthropic]
affiliation: Anthropic
arxiv:
year: 2026
read_date: 2026-04-27
tags: [context-engineering, prompt-cache, agent-architecture]
related: [anthropic2026-token-counting, anthropic2026-prompt-caching, anthropic2026-compaction, anthropic2026-context-editing, anthropic2026-context-stack, 2604.14228-dive-into-claude-code]
cited_by: []
verdict: "1M 是天花板不是默认；超 200K 部分 2x 计费这条潜规则定义了 caching/compaction/editing 的优先级——所有上层 context 工程都是在挤这个预算。"
rating: 4/5
---

# Context Windows

> **关于这份笔记**：Anthropic 平台官方文档不是研究论文，但讲的就是"预算约束"——它是其他 4 个 context 原语（caching / compaction / context-editing / token-counting）的元问题。按照仓库的习惯把它当一篇 spec-paper 处理，重点放在数字、潜规则与跨原语的依赖。

## 1. TL;DR

**Context window** = 一次请求里 **input + output + tool 调用结果 + thinking + cache** 全加在一起的 token 上限。Claude 4.X 系列把上限推到 **1M tokens**（Mythos Preview / Opus 4.7 / Opus 4.6 / Sonnet 4.6 standard pricing），Sonnet 4 也支持 1M 但要 **tier 4** 且超过 200K 部分按 **2x input / 1.5x output** 收费；旧型号上限 200K。**这个上限是其他 4 个原语的元问题**——caching 让 prefix 重读便宜、compaction 让旧消息腾位、context-editing 删 tool result、token-counting 测度量衡，**所有都是为了让任务装得进 context window**。

### 说人话

> Claude 一次能"看见"的最多就 **1M tokens**（约 75 万行代码 / 几十篇论文）。这个数字看着大，但 agent 跑工具调用、堆 tool result、装 CLAUDE.md、留对话历史，几个小时就能撑爆。所有别的"context 原语"都是在跟这个上限赛跑。

#### 术语翻译

| 原词 | 说人话 |
|---|---|
| **context window** | 模型一次能"看进去"的所有内容（**包括它自己的回答**），单位 token |
| **token** | 模型的"字"——大约 1 个英文 token = 0.75 个英文单词 = 1-3 个汉字字符 |
| **tier 4** | Anthropic 的用量等级第 4 档；要烧到一定金额或申请 custom rate limit 才能解锁 1M（针对 Sonnet 4） |
| **premium pricing** | 当请求超过 200K 时部分模型自动按更贵价格收费（input 2x、output 1.5x） |
| **remaining capacity** | 每次 tool call 后 Claude 收到的"还剩多少 token"提示——agent 用这个调整后续行为 |

#### "context window 是 working memory" 什么意思

它**不是知识库**（那是训练数据，不可变）。它是**工作内存**：你这次给它的 system prompt + 对话历史 + 工具结果 + 它正在生成的回答。**模型把这一坨当一个整体看待**——所以放 50 个无关 PDF 进去会让它对真正重要的内容"分心"，这也是 *context rot* 现象的根源。

#### 反直觉点

**1M 不是"免费午餐"**：
- Sonnet 4 的 1M 模式要付 **2x 输入 + 1.5x 输出** 溢价（高于 200K 部分）；Opus 4.7/4.6/Sonnet 4.6/Mythos Preview 是 standard pricing 直接给 1M——**模型选型直接绑定预算策略**
- Tier 4 门槛把小客户排除在 Sonnet 4 1M 之外，但 4.6 系列对所有 tier 开放——**"涨 context 不涨价"是产品差异化**

**1M 不能保护你不踩 *context rot***：
- 配套 cookbook（[[anthropic2026-context-stack]]）实测：把 8 篇 ~40K tokens 的文档全读进 1M 窗口，agent 仍能完成任务，但 *早期文档的细节召回度* 会随窗口填满而下降
- 这就是为什么 Anthropic 在博客里反复说 "context engineering 是找最小的 high-signal token 集合"——1M 给你**容错**，不是**许可**

## 2. 论文身份

- **标题**：Context windows
- **作者**：Anthropic（文档站）
- **机构**：Anthropic
- **出处**：Claude API Docs，build-with-claude 系列
- **链接**：<https://platform.claude.com/docs/en/build-with-claude/context-windows>
- **快照日期**：2026-04-27（platform.claude.com 区域屏蔽，内容来自 Anthropic 官方 cookbook 与多渠道交叉验证）

## 3. 它解决什么问题

**问题**：所有 transformer 模型推理时都有"一次看进去多少 token"的硬上限。这条上限是：
1. **架构性**的（attention 的二次方内存、KV cache 大小）
2. **经济性**的（更大窗口意味着更高 prefill 成本与延迟）
3. **统计性**的（context rot：窗口越满，召回越差）

**为什么仅用 caching / RAG 解决不够**：
- caching 不**消除** token，只让重复的 prefix **更便宜**
- RAG 不**消除** prompt 大小，把检索结果塞进去仍计入 window
- fine-tuning 把知识**写进权重**，但 ad-hoc 工具结果 / 用户 session 状态没法 fine-tune

**Context windows 文档**回答的是：
- **我有多少预算可以挥霍？**（model 决定）
- **超额了会发生什么？**（API 直接拒绝 prompt_too_long / 静默截断 / 自动 premium pricing 取决于 model）
- **怎么知道还剩多少？**（agentic 任务中每次 tool call 后的 remaining capacity broadcast）

## 4. API 机制 / 契约

### 4.1 上限按 model 决定

| Model | Context window | 备注 |
|---|---|---|
| Mythos Preview | 1M | standard pricing |
| Opus 4.7 | 1M | standard pricing |
| Opus 4.6 | 1M | standard pricing |
| Sonnet 4.6 | 1M | standard pricing |
| Sonnet 4 | 1M | **tier 4 required**；>200K 按 2x input / 1.5x output 收费 |
| Sonnet 3.7 / Opus 3 / Haiku 3.5 等 | 200K | 旧型号 |

**没有显式的 API 字段** "max_context_tokens" 让你查询；这个数字是 model name 的**隐性合同**。

### 4.2 超额行为

当 input 超过 model 的 context window：
- API 返回错误 `prompt_too_long`（HTTP 400）
- 没有自动截断、没有自动摘要——**调用方负责**

当 input 在 window 内但接近上限：
- output 会被压缩（max_tokens 参数被 model 隐式收紧）
- 如果开启 server-side compaction（见 [[anthropic2026-compaction]]）会触发自动 summary

### 4.3 Capacity awareness（agent 才有）

每次 tool call 后，Claude 收到一段 system 提示告知 remaining input token capacity。模型据此决定：
- 还能不能 read_file 一个大文件
- 是否该提前 record_finding 而不是继续 search
- 是否该 invoke memory tool 把状态外溢

**这条机制的工程含义**：agent 不是"一上来知道窗口大小"，而是"每一步都被告知剩余预算"。这与 system prompt 里硬编码 `your context is 1M tokens` 是两回事——后者会让 model 误判。

### 4.4 1M 不是无副作用

文档与社区分析共同指出 4 个隐性成本（按发生顺序）：
1. **Prefill 延迟随 window 线性增长**——每个 turn 都要重新算 KV，1M 比 200K 慢 5x
2. **Cache invalidation 代价更高**——切前缀让 200K cache 失效 vs 让 1M cache 失效，重建成本不同
3. **Context rot**（[[anthropic2026-context-stack]] 详述）——召回随 window 填满下降
4. **Token 成本**：1M @ standard 也是 1M × input price，不是免费的

## 5. 可观测的行为（cookbook 实测）

> 来自 `tool_use/context_engineering/context_engineering_tools.ipynb` 的 baseline 实验

### 5.1 1M 窗口 baseline（无 context management）

任务：让 agent 读完 8 篇 ~40K token 的研究文档（共 ~330K tokens）后做 cross-doc synthesis：

- 完整跑完，**无硬错误**
- Peak context：~340K tokens
- 但 *早期文档的细节* 在 synthesis 阶段召回明显下降——agent 能引用 batch 2 文档的细节，对 batch 1 第一篇的 appendix 表格几乎"想不起"
- 每个 turn 平均 prefill 时间：随 turn 数线性增长

### 5.2 200K 窗口同任务

- 在第 5 个 read_file 后 API 返回 `prompt_too_long`
- 任务**硬失败**——agent 没机会 graceful degrade

### 5.3 后果

这个实验直接说明：**1M 把 200K 的硬失败换成了 1M 的软失败（context rot）**。两种失败都需要上层原语来缓解——这就是为什么 [[anthropic2026-compaction]] / [[anthropic2026-context-editing]] / [[anthropic2026-prompt-caching]] 全是必要的。

## 6. 关键数字与 gotcha

### 6.1 计费表（从文档与社区交叉确认）

| Model 类 | ≤200K input | >200K 段 | output |
|---|---|---|---|
| Sonnet 4 | 1x | **2x input** | **1.5x output** |
| Opus 4.7 / 4.6 / Sonnet 4.6 / Mythos Preview | 1x | 1x | 1x |

> 所以**实际定价 = model_choice × token_count × tier_modifier**——这条三元组是 context budget 决策的真实公式。

### 6.2 反直觉

- **context window 包括 output**：max_tokens=8192 会从你的 input 预算里"预订"8192 个位子。多 turn 时累加严重
- **thinking tokens 也算**：extended thinking 模式下 thinking 输出占 window；这就是为什么有 `clear_thinking_20251015` edit type（见 [[anthropic2026-context-editing]]）
- **cache_creation_input_tokens 也算 input**：第一次 cache 写入是按 1.25x 计费（见 [[anthropic2026-prompt-caching]]），但**仍然**占 context budget——cache 是省钱不是省位子
- **count_tokens 是估值**：用 [[anthropic2026-token-counting]] 预估的数字与真实 `usage.input_tokens` 可能差几个百分点；预算贴着 1M 跑会爆

### 6.3 Tier 限制

Sonnet 4 1M 需要 **tier 4 或 custom rate limit**，意味着小账户**没有这个选项**——产品差异化是真实存在的，做产品规划时不能假设客户都能用到。

## 7. 限制

**Anthropic 文档承认（隐式）**：
- 没有公开"模型在多大 context 时性能开始下降"的曲线
- 没有 SLO 保证 capacity awareness 提示的 token 准确度（实测中是估值）
- premium pricing 的精确触发逻辑（按 prompt-level 还是 turn-level 计算）未公开

**我看出的**：
- "context window" 是个**单一 scalar**——文档没区分 system / user / assistant / tool_result 各占多少**子预算**，但这些角色的内容确实有不同的注意力权重（[[2604.14228-dive-into-claude-code]] §7.1 揭露了 CLAUDE.md 是 user message 不是 system prompt 的设计选择）
- 文档没讲**1M 窗口下的 prefill latency 数字**，但 cookbook 实测显示是显著的——这条会被忽略，但对低延迟交互式产品（IDE / chat）至关重要
- 没有说"1M tokens 究竟用什么 tokenizer"——客户端用 tiktoken 等估值会偏差，应该用 [[anthropic2026-token-counting]] 的 `count_tokens`

## 8. 与我关心的问题的联系（context assemble / compact / cache）

### 8.1 这是预算约束，不是工具

context-windows 本身**不是一个可调用的 API**，而是"你买了多少 context 预算"的合同。所有别的原语都是在花这个预算。理解这点后，managed-agent 系统设计的核心问题变成：

> **"我的预算花在哪 4 类 token 上？"**
> 1. system + 不变 prefix（用 caching 摊销 → 见 [[anthropic2026-prompt-caching]]）
> 2. 累积 tool result（用 clearing 删除 → 见 [[anthropic2026-context-editing]]）
> 3. 长程对话历史（用 compaction 摘要 → 见 [[anthropic2026-compaction]]）
> 4. 跨 session 状态（用 memory tool 外溢 → cookbook context_engineering）

### 8.2 1M ≠ 应该全部用满

cookbook 给出的明确建议（与 Anthropic context engineering blog 一致）：
> "the core discipline is finding the smallest set of high-signal tokens that maximize the likelihood of your desired outcome"

managed-agent 工程化要点：
- **不要把 1M 当目标**，把它当**容错冗余**
- **每个 task 类型设计自己的 sweet spot**：长 QA 30-50K 足够，长程 agent 200-300K 已经偏多
- 监控 **context rot**（用 needle-in-haystack probe 测召回）作为质量指标

### 8.3 对 Claude Code 5 层 compaction pipeline 的映射

[[2604.14228-dive-into-claude-code]] 描述 Claude Code 把 1M context window 当作硬约束，建了 5 层 compaction pipeline 防溢出。这映射到 API 原语：

| Claude Code 层 | 等价 API 原语 |
|---|---|
| Budget reduction（per-tool-result 限长） | 客户端 only（API 不直接提供） |
| Snip（older history trim） | 客户端 only |
| Microcompact（cache-aware） | ≈ [[anthropic2026-context-editing]] `clear_tool_uses_20250919` |
| Context collapse（read-time projection） | **API 没有等价**——这是 Claude Code 独家发明 |
| Auto-compact（LLM summary） | ≈ [[anthropic2026-compaction]] `compact_20260112` |

**结论**：context-windows 这条预算约束在 API 侧给了 3 个原语，**剩下 2 个（budget / snip / collapse）必须客户端自己实现**。设计 managed-agent 时要清楚这个职责分工。

### 8.4 不该套的场景

- **极短交互**（<10K tokens 单 turn）：所有 context 工具都是浪费精力，甚至可能因 caching min-length 不达标而无效
- **数据科学 pipeline**（一次性巨量输入 → 单次输出）：1M 模型直接处理比上工具链便宜
- **强一致性审计场景**：context rot 不可量化，1M 窗口下"模型说它读过"≠"模型真的能复述"——要么用更小窗口 + 强制召回 probe，要么直接 RAG

---

## 引用与相关

- 原文：<https://platform.claude.com/docs/en/build-with-claude/context-windows>（区域屏蔽，未直接抓到；通过 Anthropic cookbook + 多源交叉验证）
- 镜像（也屏蔽）：<https://docs.anthropic.com/en/docs/build-with-claude/context-windows>
- Anthropic 官方 cookbook（GitHub 可拉）：[`tool_use/context_engineering/context_engineering_tools.ipynb`](https://github.com/anthropics/claude-cookbooks/blob/main/tool_use/context_engineering/context_engineering_tools.ipynb)
- Anthropic blog：[Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- 1M launch：[Claude Sonnet 4 now supports 1M tokens of context](https://www.anthropic.com/news/1m-context)
- Context rot 实证：[research.trychroma.com/context-rot](https://research.trychroma.com/context-rot)
- Pricing：<https://platform.claude.com/docs/en/about-claude/pricing>
- 同 stack 其他原语：[[anthropic2026-token-counting]] [[anthropic2026-prompt-caching]] [[anthropic2026-compaction]] [[anthropic2026-context-editing]]
- 综述：[[anthropic2026-context-stack]]
- 实现侧对照：[[2604.14228-dive-into-claude-code]]
- 我从哪里看到引用：用户（context engineering 研究方向）
