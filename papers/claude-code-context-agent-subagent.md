---
title: "Claude Code Subagent 与上下文"
authors: [Xianping]
affiliation: managed-agents-sphere
arxiv:
year: 2026
read_date: 2026-04-29
tags: [agent-architecture, context-engineering, prompt-cache, memory]
related: [anthropic2026-context-engineering, anthropic2026-context-stack, 2604.14228-dive-into-claude-code]
cited_by: []
verdict: "Claude Code 源码级 subagent 路径剖析（named vs fork）、5 段独立通道、cache 共享性；与 2604.14228 公开论文版互补——后者讲设计空间、本文讲实现锚点"
rating: "—"
---

# Claude Code Subagent 与上下文

> 第一手材料：`../claude-code/src/`（55 个目录的 TypeScript 源码）。
>
> 范围：subagent 调用路径、上下文组装机制、cache 行为、与 2604.14228 公开论文的差异。
>
> 引用风格：仅 file:line 锚点 + 短代码片段（≤10 行），不含 verbatim 大段源码。

---

## 概览

### TL;DR

1. **subagent 有两条独立路径**：
   - **named**（显式 `subagent_type`）—— 走 `runAgent`，独立 cache 命名空间，独立 system prompt + 工具池
   - **fork**（隐式，`subagent_type` 缺省 + 实验开）—— **byte-exact 继承父对话前缀**，专为 prompt cache 复用而设计
2. **AgentTool.call() 一次 → 4 路 dispatch**：`teammate / remote / async / sync`（sync 可中途升格 background）
3. **中央构造函数** `createSubagentContext` (`utils/forkedAgent.ts:345`) 一次性决定 32 个字段隔离 / 共享。默认全隔离 + 5 个 opt-in 共享 + 1 个永远穿透 + 1 个永远共享
4. **5 段独立通道**：`systemPrompt + userContext + systemContext + options + messages` 各自决策。三个 context 段是 query() 独立参数，**不是塞进 system prompt**
5. **CLAUDE.md 默认进 child；MEMORY.md（auto memory）默认不进**——后者要 `agentDef.memory` 显式声明，6 个内建都未启用
6. **Explore / Plan 是「轻量 read-only」**：`omitClaudeMd: true` + 丢 gitStatus，节省 ~5–18 Gtok/week
7. **Cache 关系**：同 agentType 的 sibling 共享前缀；不同 agentType 各自独立；fork sibling 全部共享父前缀，仅末尾 directive 文本 diverge
8. **递归 fork 硬禁止**：`querySource === 'agent:builtin:fork'` 或消息含 `<fork-boilerplate>` tag → throw（compaction 仍能识别）
9. **`subagent_type` schema 是 `z.string().optional()`**——不是 enum；agent 列表通过 attachment message 注入（`agent_listing_delta`），节省 ~10.2% fleet cache_creation tokens
10. **child → parent 返回**：sync 路径塞 last assistant text 到 tool_result；async 路径返 `{agentId, outputFile}`，完成时 `<task-notification>` 走父下轮 messages
11. **`isConcurrencySafe() === true`**：父同 turn 可并发起多个 Agent，每个独立 agentId / queryTracking.chainId / readFileState clone / hooks scope / `todos[agentId]`
12. **6 处 PLAN A 修订**（`subagent_type` 描述、subagent cache 共享性、MEMORY.md 注入条件、Explore/Plan 量化、SubagentStop 转换、tool description attachment 优化）

---

## 1. 源码定位

### 1.1 顶层目录索引（精简）

> 「★」= subagent 上下文组装路径必经；其余按职能分组。

| 路径 | 职能 |
|------|------|
| `Task.ts` ★（125） | 7 类 TaskType + ID 生成 |
| `tasks.ts` ★（39） | task 注册表 |
| `Tool.ts` ★（792） | Tool 类型 + `buildTool` 工厂 |
| `tools.ts` ★（389） | 工具注册表 + `assembleToolPool` |
| `query.ts` ★★★（1729） | **主查询循环**——主对话 / named / fork 共用 |
| `context.ts` ★（189） | `getSystemContext` + `getUserContext`（memoized） |
| `tools/` ★★★ | 38 个工具，含 `AgentTool/`（13 文件） |
| `tasks/` ★★ | 6 类 task 实现（LocalAgent / Remote / Shell / InProcessTeammate / Dream / Workflow） |
| `utils/` ★★★ | `forkedAgent.ts`（689）/ `agentContext.ts`（178）/ `messages.ts`（5512）/ `hooks/` |
| `services/` | analytics / api / mcp / compact / SessionMemory / AgentSummary |
| `coordinator/` | `coordinatorMode.ts`（多 agent 编排，与 fork 互斥） |
| `commands/` | 100+ 子目录，每个一个 slash command |
| `skills/` | bundled / loadSkillsDir / mcpSkillBuilders |
| `memdir/` | MEMORY.md 系统（findRelevantMemories / memoryScan / paths） |
| `bootstrap/state.ts` | 1758 行——全局可变态枢纽 |
| `entrypoints/` | `cli.tsx` / `init.ts` / `mcp.ts` / `sdk/` |
| `hooks/`（88 个 use*.ts） | **React hooks**（UI 层） |
| `utils/hooks/` | **配置型 hooks 系统**（PreToolUse / PostToolUse 等） |
| `bridge/` `buddy/` `cli/` `dialogLaunchers.tsx` `replLauncher.tsx` `main.tsx` | 入口与 UI 桥接 |
| `components/` `screens/` `ink/` `vim/` `voice/` | UI / 终端渲染 |

⚠ **两套同名 `hooks/`**：`src/hooks/` 是 React hooks（UI）；`src/utils/hooks/` 才是 PreToolUse/PostToolUse 那套。新人极易混淆。

### 1.2 5 维主题 → 文件映射

| 维度 | 关键文件 |
|------|----------|
| **02 Context 组装** | `context.ts` / `utils/forkedAgent.ts:345` / `utils/messages.ts` / `utils/claudemd.ts` / `memdir/` / `utils/attachments.ts` |
| **03 Cache** | `services/api/promptCacheBreakDetection.ts` / `utils/forkedAgent.ts`（CacheSafeParams）/ `tools/AgentTool/prompt.ts:48-66` / `services/compact/` |
| **04 Tools / Hooks** | `Tool.ts` / `tools.ts` / `tools/*/` / `utils/hooks/hookEvents.ts` / `utils/hooks/execAgentHook.ts` |
| **05 MCP** | `services/mcp/{client,config,types}.ts` / `tools/AgentTool/runAgent.ts:95-218`（agent-specific MCP） |
| **06 Skills** | `skills/{bundled,loadSkillsDir,mcpSkillBuilders}.ts` / `commands.ts:getSkillToolCommands` / `utils/forkedAgent.ts:191-232`（skill 走 fork） |

### 1.3 subagent 直系核心文件

**`tools/AgentTool/`**（13 文件）：

| 文件 | 行 | 作用 |
|------|----|------|
| `AgentTool.tsx` | 1397 | 工具定义、`call()` 4 路 dispatch |
| `runAgent.ts` | 973 | **named subagent 主循环**——MCP 初始化、tool pool、调 `query()` |
| `forkSubagent.ts` | 210 | fork 专用：`FORK_AGENT` 合成、`buildForkedMessages`、`isInForkChild`、worktree notice |
| `prompt.ts` | 287 | tool description；`shouldInjectAgentListInMessages()`（cache 优化） |
| `agentMemory.ts` | 177 | per-agent MEMORY.md 加载 |
| `agentMemorySnapshot.ts` | — | memory 快照 |
| `agentToolUtils.ts` | — | `resolveAgentTools` / `runAsyncAgentLifecycle` / `finalizeAgentTool` 等 |
| `loadAgentsDir.ts` | — | 加载 `~/.claude/agents/*.md` 与项目级定义 |
| `builtInAgents.ts` | 72 | 6 个 built-in 注册 + gating |
| `built-in/` | — | 6 个内建定义子目录 |
| `resumeAgent.ts` | — | 恢复后台 agent（结合 sidechain transcript） |
| `constants.ts` `agentColorManager.ts` `agentDisplay.ts` `UI.tsx` | — | 常量 / 渲染辅助 |

**`utils/forkedAgent.ts`（689）★★★**——subagent 上下文组装的中央位置：

- `CacheSafeParams`（L57-68）：父子必须 byte-exact 的 5 元组
- `createSubagentContext()`（L345-462）：所有 subagent 都过它
- `runForkedAgent()`（L489-626）：fork 专用查询循环
- `prepareForkedCommandContext()`（L191-232）：skill / slash command 走 fork 的预处理

**`tasks/LocalAgentTask/LocalAgentTask.tsx`（682）**：异步 subagent 任务包装器；`registerAsyncAgent / completeAsyncAgent / failAsyncAgent / killAsyncAgent`。

**`Task.ts`（125）**：7 类 TaskType（`local_bash / local_agent / remote_agent / in_process_teammate / local_workflow / monitor_mcp / dream`），ID 前缀 `b a r t w m d`，36^8 ≈ 2.8 万亿组合防 symlink 暴破。

**`context.ts`（189）**：仅 2 个 memoized 函数——`getSystemContext()`（gitStatus + 可选 cacheBreaker）、`getUserContext()`（claudeMd + currentDate）。subagent 复用父的 memoized 结果。

---

## 2. AgentTool 的 4 路 dispatch

### 2.1 路径选择逻辑

`AgentTool.call()` (`AgentTool.tsx:239-1262`) 按下面顺序选 1 路：

```
1. teammate spawn:    if (team_name && name) → spawnTeammate
2. remote_launched:   if (effectiveIsolation === 'remote' && build === 'ant') → teleportToRemote
3. async_launched:    if (shouldRunAsync) → registerAsyncAgent + runAsyncAgentLifecycle
4. sync foreground:   else → runAgent inline，可中途升格 async（BackgroundPromise race）
```

`shouldRunAsync` 触发条件：`run_in_background: true` / `agentDef.background: true` / `coordinator mode` / `fork experiment` / `KAIROS assistant mode` / `proactive active`，且 `!isBackgroundTasksDisabled`。

### 2.2 调用链骨架

```
parent assistant emits tool_use=Agent
        │
        ▼
AgentTool.call()
    │  ├─ 解析 effectiveType = subagent_type ?? (forkEnabled ? undefined : 'general-purpose')
    │  ├─ 加载 selectedAgent（FORK_AGENT 或 activeAgents 中查找；filterDeniedAgents 过滤）
    │  ├─ workerTools = assembleToolPool(workerPermissionContext, mcpTools)  ← 与父 mode 解耦
    │  ├─ 检查 isolation: 'worktree' / 'remote'
    │  └─ 4 路分流：
    │
    ├──► teammate:  spawnTeammate({...})  → in-process / tmux roster
    ├──► remote:    teleportToRemote + registerRemoteAgentTask  → CCR
    ├──► async:     registerAsyncAgent + runAsyncAgentLifecycle(makeStream: ()=>runAgent({...,override:{agentId, abortController}}))
    │                  返回 {status:'async_launched', agentId, outputFile}
    └──► sync:      runAgent({...,override:{agentId}})[Symbol.asyncIterator]
                       ├─ Promise.race([next, backgroundPromise])  ← 2s 后显示 BackgroundHint
                       ├─ 若 backgrounded：转入 async 路径继续
                       └─ 否则：finalizeAgentTool → tool_result {status:'completed', content, ...}
```

三条本地路径（async / sync / fork）**都最终 call `runAgent()`**；fork 是 named 路径的特例（`useExactTools=true`、`forkContextMessages=parent.messages`、`override.systemPrompt=父的渲染后字节`）。

### 2.3 4 路 dispatch 各自做什么

> 本节回答「每条路径**为什么存在**、**典型用例**」；下一节 §2.4 从生命周期细节做对照。

#### ① teammate spawn — 多 agent 团队（双向通信）

- **触发**：参数同时带 `team_name + name`
- **本质**：起一个**有名字的队友**，能被父用 `SendMessage({to: name})` 主动喊话
- **关键**：是 4 路里**唯一双向通信**的——其他 3 路都是「父 → 子单向，子完成返回结果就完」
- **用例**：多 agent 协作（写代码 + review 互相讨论）、长期常驻助手
- **底层**：`spawnTeammate` 起 in-process 或 tmux teammate，注册 `agentNameRegistry`（name → agentId）

#### ② remote_launched — 远端 CCR 环境（仅内部）

- **触发**：`effectiveIsolation === 'remote'` + `external === 'ant'`
- **本质**：把 agent 任务发到**云端另一台机器**跑——不在本机
- **关键**：**唯一跨进程跨机器**的路径；外部 build 整段 dead-code-eliminate
- **用例**：跑潜在危险代码（隔离）、资源密集任务、需要专门 sandbox
- **底层**：`teleportToRemote` 上传 bundle → CCR 起 session → `registerRemoteAgentTask` 拿 taskId/sessionUrl，父立即返回（async 远端版）

#### ③ async_launched — 后台异步

- **触发**（任一）：`run_in_background:true` / `agentDef.background:true` / fork 实验 / coordinator / KAIROS / proactive
- **本质**：父立即拿 `{agentId, outputFile}` 走人，子在**同进程后台**跑；完成时 `<task-notification>` 进父下轮 messages
- **关键**：同进程 async function，但 `abortController` 不挂父——父按 ESC 子继续跑（明确 kill 才停）
- **用例**：长任务（编译、大重构）、用户 prompt 写后台执行、fork 实验下默认全异步

#### ④ sync foreground — 同步前台（默认）

- **触发**：以上 3 个都不命中
- **本质**：在父 turn 内 `await` 等子跑完，子的 last assistant text 直接塞回父 tool_result
- **关键**：唯一**会动态升格**的路径——跑 >2 秒 UI 弹「可后台化」hint，用户接受后前台 iterator 关、新建后台 iterator 接着跑
- **用例**：默认情况；模型自己 spawn `Explore` 让它搜代码再继续
- **底层**：`Promise.race([agentIterator.next(), backgroundPromise])`，`abortController` 跟父共享（父 ESC 子一起挂）

#### 4 路对比

| 维度 | teammate | remote | async | sync |
|------|----------|--------|-------|------|
| 进程位置 | 本机同进程 / tmux | 远端机器 | 本机同进程 | 本机同进程 |
| 父等不等 | 不等（持续运行） | 不等 | 不等 | 等 |
| 父 → 子通信 | **双向**（SendMessage） | 单向 | 单向 | 单向 |
| 子 → 父返回 | 异步消息（持续） | 完成通知 | 完成通知 | tool_result（一次） |
| abortController | 各自管理 | 各自管理 | 新建不挂父 | 共享父 |
| 触发关键字 | `name + team_name` | `isolation:'remote'` | `run_in_background` 等 | 默认 |
| 可见性 | 内外都有（KAIROS gated） | 仅内部（ant build） | 内外都有 | 内外都有 |

记忆口诀：

| 路径 | 类比 |
|------|------|
| teammate | 长期同事——双向、有名字 |
| remote | 外包给云端——不在本机 |
| async | 让秘书去办——不等、给个工单号 |
| sync | 站着等他办完——默认、最常见 |

### 2.4 sync vs async vs fork 关键差异

| 维度 | sync named | async named | fork |
|------|-----------|-------------|------|
| 触发 | 默认 | `run_in_background:true` 等 | `subagent_type` 缺省 + 实验开 |
| 父 turn | 阻塞等子 | 立即返回 `{agentId, outputFile}` | （走 async 路径，因 forceAsync） |
| 返回正文 | tool_result 含 last assistant text | 完成时 `<task-notification>` 进父下轮 messages | 同 async |
| abortController | 共享父的 | 新建不挂父（用户 ESC 父，子继续跑） | 同 async |
| `shouldAvoidPermissionPrompts` | false | true | true |
| `isNonInteractiveSession` | 跟父 | 强制 true | 跟父（`useExactTools` 时） |
| `shareSetAppState` | true | false | false |
| `thinkingConfig` | `{ type: 'disabled' }` | `{ type: 'disabled' }` | 跟父 |
| 工具池 | `resolveAgentTools(agentDef.tools)` | 同 | `parent.options.tools`（exact array） |
| systemPrompt | agent 自己的 + env details | 同 | `parent.renderedSystemPrompt`（byte-exact） |
| Cache 命名空间 | 独立 | 独立 | **共享父前缀** |

**sync→async 升格**：sync 跑超过 2 秒 UI 弹「可后台化」hint；用户接受后前台 iterator 关掉，新建后台 iterator 接着跑（`agentIterator.return().catch(()=>{})` 释放 finally → 重新 `runAgent({...,isAsync:true})`）。这是唯一的 path 切换点。

---

## 3. 上下文组装详解 ★

> 本节是 PLAN C 主菜。child 的「上下文」由 5 段独立通道构成，进 `query()` 时分别作为参数传入。

### 3.1 5 段独立通道

| 通道 | 内容 |
|------|------|
| `systemPrompt` | agent 定义的 prompt + 环境 details（cwd / OS / model / 工具列表 / 绝对路径与 emoji 准则） |
| `userContext` | `claudeMd` + `currentDate` |
| `systemContext` | `gitStatus`（启动快照，最大 2KB）+ 可选 `cacheBreaker`（ant-only） |
| `toolUseContext.options` | `tools[]`, `commands[]`, `thinkingConfig`, `mainLoopModel`, `mcpClients[]`, `mcpResources`, `agentDefinitions`, `isNonInteractiveSession`, `debug`, `verbose`, `appendSystemPrompt`, `querySource`(fork only) |
| `messages` | 对话历史 |

### 3.2 父 → 子继承什么

| 项 | 是否继承 | 锚点 |
|----|----------|------|
| **system prompt** | named: ❌ 重新算<br>fork: ✅ 父的 `renderedSystemPrompt` | `runAgent.ts:508-518` / `AgentTool.tsx:495-511` |
| **CLAUDE.md** | ✅ 默认<br>❌ Explore/Plan（`omitClaudeMd`） | `runAgent.ts:380-398` |
| **gitStatus** | ✅ 默认<br>❌ Explore/Plan（agentType 硬编码） | `runAgent.ts:400-410` |
| **currentDate** | ✅ 总是 | `context.ts:186` |
| **cwd** | ✅ 默认 `getCwd()`；worktree/cwd override 包整个 child | `AgentTool.tsx:638-641` |
| **env (process.env)** | ✅ 完全（同进程） | — |
| **MCP clients** | ✅ 父 + agent.mcpServers 追加 | `runAgent.ts:95-218, 648-664` |
| **配置型 hooks** | ✅ 父注册的；child 通过 wrapped getAppState 读到 | `runAgent.ts:557-575` |
| **agentDefinitions registry** | ✅ 透传 | `runAgent.ts:687` |
| **mainLoopModel** | ❌ override > agentDef.model > parent | `runAgent.ts:340-345` |
| **thinkingConfig** | named: ❌ 强制 disabled<br>fork: ✅ 父的 | `runAgent.ts:682-684` |
| **isNonInteractiveSession** | named sync: 父的 ?? false<br>named async: 强制 true<br>fork: 父的 | `runAgent.ts:668-672` |
| **fileReadingLimits / userModified** | ✅ 共享同对象引用 | `forkedAgent.ts:456-457` |
| **updateAttributionState** | ✅ 永远共享（functional + scoped 安全） | `forkedAgent.ts:434-435` |
| **toolPermissionContext.alwaysAllowRules.cliArg** | ✅ 总是保留 SDK `--allowedTools` | `runAgent.ts:472-477` |

### 3.3 CLAUDE.md vs MEMORY.md 在 subagent 里的处理

> 两个「记忆」入口在 child 里命运完全不同。

| 入口 | 默认在 child 里 | 例外 / 配置 |
|------|----------------|-------------|
| **CLAUDE.md（8 层级联）** | ✅ 进 `userContext.claudeMd` | `omitClaudeMd: true` 跳过——Explore / Plan 默认开（GrowthBook `tengu_slim_subagent_claudemd` 默认 true，节省 5–15 Gtok/week 跨 34M+ Explore spawn） |
| **MEMORY.md（auto memory）** | ❌ 默认**不**进 | 仅当 `agentDef.memory: 'user' \| 'project' \| 'local'` 时由 `loadAgentMemoryPrompt` 注入 child system prompt（agent 自己的 `getSystemPrompt` 内调用） |
| **`<system-reminder>`** | child 的 SubagentStart hook additionalContexts 以 attachment 注入 | — |
| **memdir prefetch** | child 调 `query()` 时同样跑 `startRelevantMemoryPrefetch`，但读 child 自己 messages 上下文 | — |

**6 个内建 agent 都未声明 `memory` 字段** → 内建 subagent 默认无 auto memory。要 user / project / plugin agent 显式 `memory: <scope>` 才走。当前会话主对话的 auto memory（`~/.claude/projects/.../memory/`）属于主 agent，**不会自动给 subagent**。

每 agentType 的 MEMORY.md 路径：

| scope | 路径 |
|-------|------|
| `user` | `<memBase>/agent-memory/<agentType>/MEMORY.md` |
| `project` | `<cwd>/.claude/agent-memory/<agentType>/MEMORY.md` |
| `local` | `<cwd>/.claude/agent-memory-local/<agentType>/` 或 `$CLAUDE_CODE_REMOTE_MEMORY_DIR/projects/<root>/agent-memory-local/<agentType>/` |

`agentType` 经 `sanitizeAgentTypeForPath` 把 `:` 换 `-`（兼容 Windows + 插件命名空间）。

---

## 4. Cache × concurrency 矩阵

### 4.1 Anthropic API cache 命中条件

5 段必须 byte-exact：`system / tools / model / messages prefix / thinking`。

### 4.2 named vs fork 各段对照

| 段 | named subagent | fork subagent |
|----|---------------|----------------|
| system | ❌ agent 自己的 `getSystemPrompt` + `enhanceWithEnvDetails` | ✅ `override.systemPrompt = parent.renderedSystemPrompt`（byte-exact） |
| tools | ❌ `assembleToolPool(workerPermissionContext)` + `resolveAgentTools(agentDef.tools)` | ✅ `useExactTools=true` → `parent.options.tools` 字节相同 |
| model | 可能不同（`agentDef.model` 优先） | `model: 'inherit'` |
| messages | ❌ `[user(prompt)]`（normal）/ `[parentAssistant + placeholder results + directive]`（fork-via-AgentTool） | ✅ `forkContextMessages = parent.messages` + `buildForkedMessages` 末尾追加 |
| thinking | ❌ 强制 `{ type: 'disabled' }` | ✅ `parent.thinkingConfig` |

→ named 第一次 API call **必然 cache miss**，从头建立自己的 cache prefix。<br>
→ fork 设计就是**继承父字节**，仅末尾 directive 文本 diverge。

`buildForkedMessages` 关键设计（`forkSubagent.ts:107-169`）：
- 全部 `tool_use` block 配**统一占位** `'Fork started — processing in background'`——多并发 fork 的 tool_results 字节完全一致
- 末尾 user message 追一个 text block 含 directive——**唯一 per-child 不同处**

`forkedAgent.ts:520-523` 注释强调 fork 路径**不要** `filterIncompleteToolCalls`——让 `ensureToolResultPairing` 在下游做，跟父完全同路径。但 `runAgent` 走 fork（即 fork-via-AgentTool）**仍调** `filterIncompleteToolCalls(forkContextMessages)` (`runAgent.ts:370-372`)；区别：
- `runForkedAgent` 路径（session memory / supervisor / `/btw`）—— 不 filter
- `runAgent` 路径（含 fork-via-AgentTool）—— filter

实际差异可能不大（`buildForkedMessages` 已配 placeholder，filter 不丢任何 assistant），但路径不一致是事实，PLAN B 抓包验证（详 `plans/A-open-questions.md` #37）。

### 4.3 Concurrency 矩阵

| 场景 | 父 cache 复用度 | 兄弟 cache 共享度 |
|------|-----------------|-------------------|
| 单 named subagent | 完全独立命名空间 | — |
| N 个并发同 agentType named | 独立于父，但兄弟共享前缀（system 同 + tools 同 + model 同），diverge 在 user prompt 文本 | 高 |
| N 个并发不同 agentType named | 各自独立命名空间 | 低 |
| 单 fork subagent | 与父共享前缀至最后 assistant + 占位 tool_results | — |
| N 个并发 fork | 全部共享父前缀 + 共享占位 tool_results；diverge 仅末尾 directive 文本 | 极高 |
| fork + named 混合 | fork 共享父；named 独立 | 互不共享 |
| 嵌套 subagent (depth=2) | 同样规则递归 | — |
| async vs sync | 不影响（cache 看请求字节） | — |
| worktree isolation | systemPrompt 含 cwd（env details）→ 字节不同 → fork+worktree 实际可能 cache miss | ⚠ 待 PLAN B 验证（`plans/A-open-questions.md` #38） |

`isConcurrencySafe() === true` (`AgentTool.tsx:1273`)：父同 turn 可并发起多个 Agent。每个并发 child 各自独立：`agentId / queryTracking.chainId / readFileState clone / localDenialTracking / fresh Sets / 各自的 hooks scope / 各自的 MCP cleanup / 各自的 todos[agentId] / 各自的 background bash`。

### 4.4 Tool description cache 优化

**一句话**：把动态 agent 列表**从 tools 段搬到 messages 段**——会变的东西挪到「只影响自己」的位置。

API 请求 3 段独立缓存（system / tools / messages）。原本 AgentTool 的 description 写在 tools 段，里面塞了动态 agent 列表，列表一变（MCP 连接 / `/reload-plugins` / 权限模式切换）→ 整个 tools 段 cache miss。

**改法**：description 退化成静态文本，agent 列表以 attachment（`agent_listing_delta`）形式塞进 messages 段。

**收益**：~10.2% fleet cache_creation tokens 节省。

锚点：`tools/AgentTool/prompt.ts:48-66`；开关 `shouldInjectAgentListInMessages()` / env `CLAUDE_CODE_AGENT_LIST_IN_MESSAGES`。

---

## 5. 6 个内建 agent + FORK 合成

| agentType | 干什么（白话） | 一句话用例 | 工具 / 启用 |
|-----------|--------------|----------|------------|
| `general-purpose` | **通用兜底**——啥都能干的全工具代理 | 模糊任务，模型不知道选啥（"研究一下 X"） | `tools: ['*']`；总是开 |
| `statusline-setup` | **专门改终端状态栏** | "帮我配一下 Claude Code 状态栏显示" | 限定子集；总是开 |
| `Explore` | **只读搜代码**——能搜能看，不能改不能写 | "搜一下哪里调用了 `foo()`" / "这个 bug 跟哪些文件相关" | 限定 + `omitClaudeMd`；GrowthBook `tengu_amber_stoat` 默认开 |
| `Plan` | **只读规划**——给实现方案，不动代码 | "怎么实现 X 功能" / "这个重构影响哪些模块" | 限定 + `omitClaudeMd`；同上 gating |
| `claude-code-guide` | **Claude Code 文档专家**——会查官方文档 | "Claude Code 的 hooks 怎么配？" / "MCP server settings.json 长啥样？" | 限定（含 WebFetch / WebSearch）；非 SDK entrypoint |
| `verification` | **验证员**——检查工作有没有做完做对 | "确认这个任务真的完成了"（实验性） | 限定；GrowthBook `tengu_hive_evidence` 默认**关** |
| **`fork`（合成 FORK_AGENT）** | **父的分身**——继承父对话全部上下文，并行干活 | 父想分头探索 3 个方向 → 起 3 个 fork 同时跑 | 复制父的 tools / model / system prompt；`subagent_type` 缺省 + `feature('FORK_SUBAGENT')` + 非 coordinator + 非 SDK noninteractive |

> **直觉记忆**：前 6 个是「**专家代理**」（每个有自己专长 / 限定），fork 是「**克隆分身**」（无分工，纯并行）。

**Coordinator mode**（`CLAUDE_CODE_COORDINATOR_MODE`）：完全替换内建列表（`builtInAgents.ts:35-43`），与 fork 互斥（`forkSubagent.ts:34`）。

**Agent 注册表来源** 优先级（高 → 低，后写覆盖前写）：`flagSettings → policySettings → projectSettings → userSettings → plugin → built-in`。Managed policy 优先级最高，built-in 兜底。

**递归 fork 防护**（`AgentTool.tsx:325-335`）：fork child 保留 Agent 工具是为了 cache-identical tool defs，所以在 call() 时拒绝。双重判断：
1. 主：`querySource === 'agent:builtin:fork'`（compaction-resistant，写在 `context.options` 上）
2. 兜底：`isInForkChild(toolUseContext.messages)` 扫消息历史看 `<fork-boilerplate>` tag

`buildChildMessage` (`forkSubagent.ts:171-198`) 给 fork child 注入 10 条「不要再 fork」硬指令，开头要求 `Scope:` 字段，禁止 thinking-out-loud。

---

## 6. Compaction：三级压缩策略

Claude Code 源码里的 compaction 不是单一函数，而是按**成本递增**的三级——`services/compact/` 下分别对应 MicroCompact / SessionMemoryCompact / FullCompact。autoCompact 按 token 用量自动触发，按"先低成本后高成本"逐级回退；三级的代价、命中条件、压缩力度差距极大，工程上必须按层理解。

![Claude Code 三级压缩策略全景图](./assets/three-level-compact.png)

![autoCompact 自动检测与三级压缩流程](./assets/three-level-compact-workflow.png)

---

## 附录 A — 关键 file:line 锚点速查

按实现位置查找：

| 锚点 | 内容 |
|------|------|
| `utils/forkedAgent.ts:57-68` | `CacheSafeParams` 五元组定义 |
| `utils/forkedAgent.ts:107-169` | `buildForkedMessages`（仅 fork） |
| `utils/forkedAgent.ts:191-232` | `prepareForkedCommandContext`（skill / slash command 走 fork） |
| `utils/forkedAgent.ts:345-462` ★ | `createSubagentContext` 32 字段构造 |
| `utils/forkedAgent.ts:489-626` | `runForkedAgent` 主循环（内部 fork 用） |
| `tools/AgentTool/AgentTool.tsx:239-1262` | `call()` 4 路 dispatch |
| `tools/AgentTool/AgentTool.tsx:318-356` | dispatch 选路逻辑（`effectiveType`） |
| `tools/AgentTool/AgentTool.tsx:495-541` | fork vs named 的 systemPrompt + promptMessages 分支 |
| `tools/AgentTool/AgentTool.tsx:686-764` | async 路径（`registerAsyncAgent` + `runAsyncAgentLifecycle`） |
| `tools/AgentTool/AgentTool.tsx:765-1261` | sync 路径 + 中途升格 background |
| `tools/AgentTool/runAgent.ts:248-860` ★ | `runAgent` 主入口 + 22 步初始化 |
| `tools/AgentTool/runAgent.ts:380-410` | omitClaudeMd / drop gitStatus（Explore/Plan 优化） |
| `tools/AgentTool/runAgent.ts:412-498` | `agentGetAppState` wrapped 函数 |
| `tools/AgentTool/runAgent.ts:557-575` | `registerFrontmatterHooks(isAgent:true)`（Stop→SubagentStop） |
| `tools/AgentTool/runAgent.ts:648-664` | MCP 合并 |
| `tools/AgentTool/runAgent.ts:700-714` | `createSubagentContext` 调用 |
| `tools/AgentTool/forkSubagent.ts:32-39` | `isForkSubagentEnabled()` |
| `tools/AgentTool/forkSubagent.ts:60-71` | `FORK_AGENT` 合成定义 |
| `tools/AgentTool/forkSubagent.ts:78-89` | `isInForkChild` 防递归 fork |
| `tools/AgentTool/forkSubagent.ts:171-198` | `buildChildMessage` 10 条硬指令 |
| `tools/AgentTool/prompt.ts:48-66` | tool description cache 优化（10.2% 节省） |
| `tools/AgentTool/agentToolUtils.ts:276-357` | `finalizeAgentTool` 抽 last assistant text |
| `tools/AgentTool/builtInAgents.ts:22-72` | 6 个内建 + gating |
| `tools/AgentTool/loadAgentsDir.ts:106-165` | `BaseAgentDefinition` 类型 + 3 子类 |
| `tools/AgentTool/agentMemory.ts:138-177` | `loadAgentMemoryPrompt`（agent MEMORY.md 注入） |
| `Task.ts` | 7 类 TaskType + ID 生成 |
| `tasks.ts` | task 注册表 |
| `tasks/LocalAgentTask/LocalAgentTask.tsx` | 异步 subagent 任务包装 |
| `context.ts:155-189` | `getUserContext`（CLAUDE.md + currentDate） |
| `context.ts:113-150` | `getSystemContext`（gitStatus + cacheBreaker） |
| `query.ts:219-239` | `query()` 入口 + `queryLoop` 委托 |
| `query.ts:301-304` | `startRelevantMemoryPrefetch`（memdir prefetch） |
