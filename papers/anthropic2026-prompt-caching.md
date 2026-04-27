---
title: "Anthropic Prompt Cache 的三级阶梯：自动、显式、投机"
authors: [Anthropic]
affiliation: Anthropic
arxiv:
year: 2026
read_date: 2026-04-27
tags: [prompt-cache, context-engineering, agent-architecture]
related: []
cited_by: []
verdict: "三级阶梯共用同一条字节级 prefix 匹配契约——automatic 是默认起点；explicit 给你独立 TTL 与 4 个 breakpoint，但代价是 marker 管理变成你的事；speculative 把 cache 写入从请求生命周期里抽出来塞进用户键入的并行窗口。前两级省钱兼省时（read 0.1×、write 1.25×），第三级用 token 换 TTFT。"
rating: 4/5
---

# Anthropic Prompt Cache 的三级阶梯：自动、显式、投机

## TL;DR

Anthropic 的两篇 cache cookbook 合起来其实在讲同一件事的三个抽象层——每往下一级，开发者拿到更多控制权，也要承担更多复杂度：

1. **自动 cache**（automatic）——`messages.create()` 顶层一行 `cache_control={"type":"ephemeral"}`。Breakpoint 位置由系统决定，多轮对话里自动跟着尾巴往后挪。
2. **显式 cache**（explicit）——把 `cache_control` 直接挂在 content block 上。你拿到 4 个独立 breakpoint，可以给不同段落配不同 TTL，代价是每次新增 user message 都得自己把 marker 移到正确位置。
3. **投机 cache**（speculative）——把"什么时候写 cache"从请求生命周期里抽出来。用户还在打字、服务器闲着的几秒里，后台异步发一个 `max_tokens=1` 请求把 cache 种好。等用户真按提交，cache 已经 hot，TTFT 接近"只有 query 段要 prefill"。

每一级的存在都是因为上一级有一道天花板：自动模式只有一个 TTL、做不到 system 与对话分开缓存，于是有了显式；显式模式仍逃不过"用户首次请求必走 cold-start"的命，于是有了投机。三者**底层契约完全相同**——下一次请求的 prefix 与已存在的 cache 条目字节级一致，就跳过那段的 prefill。
> 来源：
> - [`misc/prompt_caching.ipynb`](https://github.com/anthropics/claude-cookbooks/blob/main/misc/prompt_caching.ipynb) —— 第一、二级
> - [`misc/speculative_prompt_caching.ipynb`](https://github.com/anthropics/claude-cookbooks/blob/main/misc/speculative_prompt_caching.ipynb) —— 第三级

---

## 1. 第一级：自动 cache

第一次接 cache 用这个就够。一行就完事：

```python
client.messages.create(
    model="claude-sonnet-4-6",
    cache_control={"type": "ephemeral"},   # ← 唯一改动
    messages=[{"role": "user", "content": f"<book>{book_content}</book>\n\nWhat is the title?"}],
)
```

Cookbook 在 ~187k token 的 *Pride and Prejudice* 全文上跑了三次同样的请求，对应 cache 的三种状态：

| 状态 | 触发条件 | usage 字段 | 耗时 |
|---|---|---|---|
| Baseline | 不带 `cache_control` | 只有 `input_tokens` | 完整 prefill |
| Cache write | 第一次同 prefix 调用 | `cache_creation_input_tokens ≈ 187k` | 与 baseline 接近 |
| Cache hit | 第二次同 prefix 调用 | `cache_read_input_tokens ≈ 187k` | 显著加速 |

这就是 cache 的全部故事——第一次写、之后读、prefix 不动就一直命中。

放进多轮对话里它自己会把 breakpoint 往后挪：

```python
conversation = []
for question in ["What is the title?", "Who are Mr. and Mrs. Bennet?", ...]:
    conversation.append({"role": "user", "content": question})
    response = client.messages.create(
        model="claude-sonnet-4-6",
        cache_control={"type": "ephemeral"},
        system=system_message,        # 长 prefix 放在 system
        messages=conversation,
    )
    conversation.append({"role": "assistant", "content": response.content[0].text})
```

| 第几轮 | cache 行为 |
|---|---|
| 1 | system + user₁ 写 cache |
| 2 | 截至 user₁ 命中；assistant₁ + user₂ 写 cache |
| 3 | 截至 user₂ 命中；assistant₂ + user₃ 写 cache |

第二轮起 input 几乎 100% 走 cache_read。这里只有一个 breakpoint slot 在工作，但它会自动跟着对话尾巴往前推——你不用动手。

**这个简单性也是上限**。一个 slot、一个 TTL、什么进 cache 由系统说了算——做不到给 system prompt 单独配 1h TTL、给对话部分配 5min；做不到把 system / 工具定义 / 大段文档分别独立缓存；做不到一次请求里同时锁住多段独立的大块上下文。需要这些，往下走一级。

---

## 2. 第二级：显式 cache

把 `cache_control` 从顶层搬到 content block 上：

```python
client.messages.create(
    system=[{
        "type": "text",
        "text": system_message,
        "cache_control": {"type": "ephemeral"},     # 独立缓存 system
    }],
    messages=[
        ...,
        {"role": "user", "content": [{
            "type": "text",
            "text": last_user_message,
            "cache_control": {"type": "ephemeral"}, # 显式锁住对话末尾
        }]},
    ],
)
```

切到这一层的明确信号：

- 想给不同段落配不同 TTL（system 用 1h、对话用 5min）
- 要把 system prompt / tool schema 独立缓存，不跟对话变化绑在一起
- 一个 prompt 里有多段大型上下文，需要分别缓存
- 单请求最多 4 个 explicit breakpoint，automatic 占 1 个 slot——两者**可以混用**

代价是 marker 管理变成你的事。Cookbook 演示的写法是封一个小类，每次发请求前找最新 user message、把 cache_control 挂上去：

```python
class ConversationWithExplicitCaching:
    def get_messages(self):
        last_user_idx = max(i for i, t in enumerate(self.turns) if t["role"] == "user")
        result = []
        for i, turn in enumerate(self.turns):
            if i == last_user_idx:
                result.append({"role": "user", "content": [{
                    "type": "text",
                    "text": turn["content"][0]["text"],
                    "cache_control": {"type": "ephemeral"},
                }]})
            else:
                result.append(turn)
        return result
```

每多一处显式控制，就多一个"我有没有把 marker 移对地方"的隐藏出错点。**别一上来就用显式**——除非你已经踩到了第一级解决不了的具体问题，否则简单的更不容易出 bug。

显式模式仍有一道天花板：**cache 只在请求发生之后才存在**。在面向终端用户的应用里，第一个用户的第一次提交永远是 cold start——cache 还没建，必须走完整 prefill。体感最长的延迟，恰好落在用户最敏感的那一刻。再往下走一级，处理的就是这件事。

---

## 3. 第三级：投机 cache

前两级都把 cache 写入和某次真实请求绑在一起。第三级的关键洞察：**cache 写入根本不需要等用户真的要东西**。用户在输入框里打字的那几秒，服务器是闲的；就在那个窗口里，后台用 `max_tokens=1` 的请求把同一份 prefix 发过去，让服务器照常 prefill 并写 cache。等用户真按提交时，cache 已经 hot。

一个 helper 包住"只触发 prefill、不真生成"的副作用：

```python
async def sample_one_token(client, messages):
    """触发 prefill 并写 cache；max_tokens=1 让 decode 一个 token 就停。"""
    await client.messages.create(
        messages=messages,
        model=MODEL,
        max_tokens=1,
        ...
    )
```

`max_tokens=1` 是技术下限——API 不收 0。但 1 已经够：服务器照样把整段 prefix 跑完 prefill 写 cache，只是 decode 部分到 1 个 token 就停。返回的内容毫无用处，丢就行。我们要的从来是它写进 KV cache 的那条副作用。

完整流程一句话——用户聚焦输入框时 fire 一个 warming task，用户键入并行进行，用户提交时等 warming 完成，复用同一份 prefix 发正式请求：

```python
initial_message = {
    "role": "user",
    "content": [{
        "type": "text",
        "text": LONG_PREFIX,
        "cache_control": {"type": "ephemeral"},
    }],
}

# 用户聚焦输入框：立即 fire warming
cache_task = asyncio.create_task(sample_one_token(client, [initial_message]))

# 用户键入（cookbook 用 sleep(3) 模拟）
user_question = await wait_for_user_submit()

# 用户提交：确保 warming 已完成
await cache_task

# 复用 prefix，追加 query；deepcopy 是命门
cached_message = copy.deepcopy(initial_message)
cached_message["content"].append(
    {"type": "text", "text": f"Answer the user's question: {user_question}"}
)

async with client.messages.stream(messages=[cached_message], ...) as stream:
    async for text in stream.text_stream:
        ...   # 首 token 几乎瞬时
```

`copy.deepcopy(initial_message)` 这一行不是仪式感，是 cache 命中的命门。Python 里 `initial_message["content"]` 是同一个 list 对象，不复制就追加 query——warming 任务里被序列化进 wire 的那段 prefix 跟你正式请求里的 prefix 立刻字节级不一致，cache 100% miss、warming 那笔 token 钱白花。这种 bug 看 log 看不出来（status 是 200、字段都在），只有盯 `cache_read_input_tokens` 才发现。

至于"省了多少秒"，cookbook 不报硬数字，因为依赖网络与负载。它给的是方向性结论：standard 路径下 TTFT 包含 ~150k tokens 的完整 prefill；speculative 路径下 TTFT 只剩 query 段那几十 token 的 prefill 加单 token decode——差几个数量级。

usage 字段印证流程的正确性——warming 那次返回的 response 里 `cache_creation_input_tokens ≈ 150k`，正式请求看到的是 `cache_read_input_tokens ≈ 150k`。两次合起来比 standard 模式多了一次 `0.1× × 150k` 的 cache_read 费用——**它没省钱**，只压缩了用户感知的 TTFT。

什么时候不要上第三级：脚本化批量调用（没"用户思考间隙"可并行）、prefix 高度动态的检索类应用（warming 写下去的 cache 没机会被复用）、prefix 还达不到最小阈值（下一节的硬约束）。

---

## 4. 三级共用的契约

三级实现差很多，但底下的契约**是同一件事**：

> 下一次请求的 prefix 与某条已存在的 cache 条目**字节级一致**，就跳过那段的 prefill。

围绕这条契约的几个数字三级共用：

| 维度 | 取值 |
|---|---|
| 最小可缓存长度 | 1,024 tokens（Sonnet）；4,096 tokens（Opus、Haiku 4.5） |
| Cache write 单价 | 1.25× base input |
| Cache read 单价 | 0.1× base input |
| 默认 TTL | 5 分钟，命中刷新 |
| 1h TTL | 写单价升至 2× base input |
| 单请求 breakpoint 上限 | 4 个 explicit；automatic 占 1 个 slot |

由此推出三件**永远不要踩**的事：

- **Prefix 短于阈值 → `cache_control` 静默无效**。不会报错、也不会缓存——只是接下来全 miss。第一件事永远是用 token counting 确认 prefix 够长。
- **Prefix 字节级不一致 → 100% miss**。timestamp、空白、字段顺序、tool schema 的任何变动都让命中归零。`copy.deepcopy` 之类的防 mutation 习惯应该长在指尖。
- **5 分钟 TTL 内必须再次命中**——否则 cache 已被 evict，下一次又是 cold start。多轮对话间隔可能超 5 分钟时考虑 1h TTL：写贵 1.6×（1.25× → 2×），但只要后续命中两三次以上就赚回来。

---

## 5. 一个被两本 cookbook 双关使用的字段：timestamp

两本 cookbook 都往 prefix 里塞 timestamp，但**目的相反**——这是只有读两本才看得出来的细节：

| 出处 | timestamp 解决什么 |
|---|---|
| `prompt_caching.ipynb` | 重跑 notebook 时撞到上次的 cache 残留——`TIMESTAMP = int(time.time())` 让每次启动 baseline / write / hit 三组对照都拿到一段独立的 prefix |
| `speculative_prompt_caching.ipynb` | 多个 session 共享同一段 prefix——用户 A 写下的 cache 不应被用户 B 命中 |

工程上的反推：

- prefix 是**公共资源**（产品手册、API 文档、开源代码）→ **不要**加 timestamp，让多 session 共享 cache，命中率最大化
- prefix 是**用户私有数据**（个人代码库、个人邮件、客户数据）→ **必须**加 timestamp 或别的隔离标识

同一个字段、两种相反方向的用法。看到 cookbook 里塞 timestamp 别条件反射地抄——先想清楚你的 prefix 属于哪一种。

---

## 6. 上手清单

按三级阶梯的顺序：

1. **从自动开始**。在已有的 `messages.create()` 顶层加一行 `cache_control={"type": "ephemeral"}`。跑两次同 prefix 请求，确认第二次 `response.usage.cache_read_input_tokens` 接近 prefix 长度——这一步过了，cache 就在工作。
2. **遇到具体痛点再切显式**。要分 TTL、要独立缓存 system prompt 或 tool schema、要细控制就上。否则别上——marker 管理是隐藏的出错来源。
3. **用户路径里有思考间隙才考虑投机**。聚焦输入框时 `asyncio.create_task(sample_one_token(...))`、提交时 `await` 等它完成、`copy.deepcopy` 后追加 query。每次正式请求结束盯一眼 `cache_read_input_tokens`：接近 prefix 长度 → warming 命中、TTFT 真的下来了；远小于或为 0 → warming 失败或 prefix 不一致，回去查。

到这一步，三级阶梯就走完了。再往后更激进的玩法（cache 跨服务共享、warming 对私有 prefix 的隔离策略、多 breakpoint 的精细 TTL 编排），都是这条契约之上的事，不是 cache 本身的故事。

---

## 引用

- [`misc/prompt_caching.ipynb`](https://github.com/anthropics/claude-cookbooks/blob/main/misc/prompt_caching.ipynb) —— 第一、二级
- [`misc/speculative_prompt_caching.ipynb`](https://github.com/anthropics/claude-cookbooks/blob/main/misc/speculative_prompt_caching.ipynb) —— 第三级
