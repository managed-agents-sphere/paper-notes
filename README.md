# paper-notes

`managed-agent-sphere` 下的论文阅读与分析归档。目的：**为以后回顾服务**——读完一篇就用统一模板写一份分析，半年后能快速找回当时的判断与上下文。

## 结构

```
paper-notes/
├── README.md                        # 索引（本文件）
├── templates/
│   └── paper.md                     # 标准模板（八段式）
└── papers/
    └── <arxiv-id>-<short-slug>.md   # 一篇论文一个文件
```

阶段一：纯 markdown，扁平存放，GitHub 直接渲染。
未来阶段：必要时叠 [Quartz 4](https://github.com/jackyzha0/quartz) 渲染成静态站，激活双链/反向链接/图谱。Obsidian 风格的 `[[wiki-link]]` 现在就可以写——GitHub 会当普通文本，Quartz 接管时自动变跳转。

## 命名规范

- **文件名**：`<arxiv-id>-<short-slug>.md`，例如 `2501.03276-commer.md`
- **slug**：论文短名，全小写、连字符分隔
- 非 arxiv 论文：用 `<venue><year>-<slug>.md`，例如 `neurips2024-icae.md`

## frontmatter 字段约定

每篇笔记顶部必须有 YAML frontmatter，字段如下：

```yaml
---
title: "<论文完整标题>"
authors: [<author 1>, <author 2>, ...]
affiliation: <机构>
arxiv: <id>                  # 可选，非 arxiv 论文留空
year: <YYYY>
read_date: <YYYY-MM-DD>      # 我读完的日期
tags: [<tag1>, <tag2>, ...]  # 见下方受控词表
related: [<slug1>, ...]      # 同 repo 内相关笔记的 slug
cited_by: [<source>, ...]    # 我从哪里看到引用了这篇（如某 blog、某论文）
verdict: "<一句话判断>"       # 半年后只看这一行就能想起来
rating: <1-5>/5
---
```

## tag 受控词表

为了未来横向聚合（"context engineering 这条线我读过哪些"），tag **必须**从下表里选；新主题加入前先 PR 更新本表。

| tag | 含义 |
|---|---|
| `prompt-compression` | 把长 prompt 压成 soft prompt / 自然语言摘要 / 潜变量 |
| `soft-prompt` | 可训练的 latent embedding（gist/ICAE/ComMer 等） |
| `context-engineering` | 上下文构造、压缩、记忆管理（agent 视角） |
| `memory` | 长期记忆 / episodic / semantic memory |
| `personalization` | LLM 用户级个性化 |
| `agent-architecture` | agent 系统设计（planner / sub-agent / tool 调用拓扑） |
| `prompt-cache` | KV cache 复用、cache hit rate 优化 |
| `rag` | retrieval-augmented generation |
| `peft` | LoRA / adapter / 参数高效微调 |
| `evaluation` | benchmark / 评测方法学 |

## 分析模板（八段式）

每篇论文按以下顺序写，便于横向比较：

1. **TL;DR** —— 技术口吻的 2-3 句话总结，**内嵌"说人话"子节**：大白话版 + 术语翻译表 + 类比，让同行和外行第一眼都能进入
2. **论文身份** —— 标题/作者/机构/出处/时间
3. **问题与动机** —— 它要解决什么、为什么现有方法不够
4. **方法** —— 架构与训练（图 + 关键超参）
5. **实验** —— 数据集 / baseline / 指标
6. **关键结果** —— 数字 + 趋势 + 反直觉发现
7. **限制** —— 作者承认的 + 我自己看出的
8. **与我关心的问题的联系** —— 最重要，回顾时只看这一段也行

## 论文索引

| ID | 标题 | 标签 | 评分 | 一句话判断 |
|---|---|---|---|---|
| [anthropic2026-speculative-caching](papers/anthropic2026-speculative-caching.md) | Speculative Prompt Caching：把 cache_creation 延迟藏在用户打字时间里 | `prompt-cache` `context-engineering` `agent-architecture` | 4/5 | 用户键入时后台 max_tokens=1 异步预热 cache，提交时 TTFT 接近无 prefill；warming 命中率 ≥80% 才划算 |
| [anthropic2026-context-stack](papers/anthropic2026-context-stack.md) ⓘ | Anthropic Context Stack：从 token 预算到跨会话记忆的五件套（合订本） | `context-engineering` `prompt-cache` `memory` `agent-architecture` `evaluation` | 5/5 | 5 件 API 原语 + 综合分析合订本；逐节拆解 context-windows/token-counting/prompt-caching/compaction/context-editing 并拼出依赖图与 4 个工作流食谱 |
| [2604.14228-dive-into-claude-code](papers/2604.14228-dive-into-claude-code.md) | Dive into Claude Code: The Design Space of Today's and Future AI Agent Systems | `context-engineering` `agent-architecture` `prompt-cache` `memory` | 5/5 | Claude Code v2.1.88 源码级架构教科书；最值得抄的是『五层 compaction pipeline』和『context collapse 是 read-time projection』 |
| [2501.03276-commer](papers/2501.03276-commer.md) | ComMer: Compressing and Merging User Data for Personalization | `prompt-compression` `soft-prompt` `personalization` `memory` | 4/5 | 风格类记忆可压；事实类记忆不可压（结构性限制） |

> ⓘ 标记为合订本/综述（不严格遵循八段式；包含 5 篇 build-with-claude 文档的逐节分析 + 跨篇综合）。

---

最近添加：
- 2026-04-27 [[anthropic2026-speculative-caching]] — cache 进阶模式：用户键入时预热 cache 降 TTFT
- 2026-04-27 [[anthropic2026-context-stack]] — Anthropic Context API 五件套合订本（与 Claude Code paper 配套阅读）
- 2026-04-26 [[2604.14228-dive-into-claude-code]] — context assemble / compact / cache 的工程范式参考
- 2026-04-26 [[2501.03276-commer]] — Anthropic context engineering 引用文献
