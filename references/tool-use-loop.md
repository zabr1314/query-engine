# Tool-Use 循环的完整实现

> 基于 `src/QueryEngine.ts` (1295行) 和 `src/query.ts` (1729行) 源码分析

## 概述

Claude Code 的核心不是一个简单的"发请求-收响应"循环，而是一个**异步生成器驱动的状态机**。从用户输入到最终输出，整个链路经过了消息预处理、流式 API 调用、tool-use 解析、权限检查、工具执行、错误恢复等十几个环节。这篇文章将带你走完这条完整的链路。

## 架构总览

```
用户输入
  │
  ▼
QueryEngine.submitMessage()          ← 入口：异步生成器
  │
  ├── processUserInput()             ← 解析斜杠命令、预处理
  ├── fetchSystemPromptParts()       ← 组装 system prompt
  ├── buildSystemInitMessage()       ← yield 系统初始化消息
  │
  └── query()                        ← 核心循环（query.ts）
       │
       ├── while (true) {
       │     ├── 微压缩 / Snip / 上下文压缩
       │     ├── 自动压缩检查
       │     ├── callModel()         ← 流式 API 调用
       │     ├── 工具执行（StreamingToolExecutor）
       │     ├── Stop hooks
       │     ├── Token budget 检查
       │     └── 继续下一轮 || 返回
       │   }
       │
       ▼
   yield SDKMessage[]                ← 流式产出结果
```

## 一、QueryEngine：对话生命周期的拥有者

`QueryEngine` 类（`QueryEngine.ts:92`）是整个对话的"状态容器"。它封装了：

- `mutableMessages` — 对话历史（跨轮次持久化）
- `abortController` — 中断控制
- `permissionDenials` — 权限拒绝记录
- `totalUsage` — 累计 token 用量
- `readFileState` — 文件读取缓存

关键设计：**一个 QueryEngine = 一个对话**。每次 `submitMessage()` 是同一对话中的一轮。

```typescript
// QueryEngine.ts:56-63
export class QueryEngine {
  private config: QueryEngineConfig
  private mutableMessages: Message[]
  private abortController: AbortController
  private permissionDenials: SDKPermissionDenial[]
  private totalUsage: NonNullableUsage
  private readFileState: FileStateCache
  // ...
}
```

## 二、submitMessage() 的完整流程

`submitMessage()` 是一个 `async *` 生成器（`QueryEngine.ts:94`）。它不是简单地 return 一个结果，而是**逐个 yield 消息片段**——这让调用方（REPL / SDK）可以实时渲染中间状态。

### 阶段 1：准备阶段

```typescript
// QueryEngine.ts:130-136 — 包装 canUseTool 以追踪权限拒绝
const wrappedCanUseTool: CanUseToolFn = async (...) => {
  const result = await canUseTool(...)
  if (result.behavior !== 'allow') {
    this.permissionDenials.push({ tool_name, tool_use_id, tool_input })
  }
  return result
}
```

然后是 prompt 组装（`QueryEngine.ts:142-165`）：

```typescript
const { defaultSystemPrompt, userContext, systemContext } =
  await fetchSystemPromptParts({
    tools, mainLoopModel, additionalWorkingDirectories,
    mcpClients, customSystemPrompt,
  })
```

这里 `fetchSystemPromptParts()` 并行执行三件事：
1. 构建 system prompt（`getSystemPrompt()`）
2. 获取 user context（CLAUDE.md + 日期）
3. 获取 system context（git 状态）

### 阶段 2：用户输入处理

```typescript
// QueryEngine.ts:206-216 — 处理斜杠命令和用户输入
const { messages, shouldQuery, allowedTools, model, resultText } =
  await processUserInput({
    input: prompt, mode: 'prompt',
    context: processUserInputContext,
    messages: this.mutableMessages,
  })
this.mutableMessages.push(...messagesFromUserInput)
```

`processUserInput()` 做了两件重要的事：
- **解析斜杠命令**（如 `/clear`、`/compact`、模型切换）
- **决定是否需要调用 API**（`shouldQuery`）。如果是纯本地命令（如 `/help`），直接返回结果，不进 query 循环。

### 阶段 3：系统初始化消息

在进入 query 循环前，yield 一个 `system_init` 消息（`QueryEngine.ts:267-278`）：

```typescript
yield buildSystemInitMessage({
  tools, mcpClients, model, permissionMode,
  commands, agents, skills, plugins, fastMode,
})
```

这是一条特殊的元数据消息，告诉前端：使用了哪些工具、MCP 服务器、当前模型等。

### 阶段 4：进入 query 循环

```typescript
// QueryEngine.ts:341-345
for await (const message of query({
  messages, systemPrompt, userContext, systemContext,
  canUseTool, toolUseContext, fallbackModel, querySource,
  maxTurns, taskBudget,
})) {
  // ... 处理每条消息
}
```

`query()` 函数（`query.ts`）是整个系统的心脏。它是一个 while(true) 循环，每次迭代 = 一轮 API 调用 + 工具执行。

## 三、query() 内部：流式 API 调用与 tool-use 循环

### 单轮迭代的生命周期

```
while (true) {
  ┌─────────────────────────────────────┐
  │  1. 上下文压缩预处理                 │
  │     ├── 微压缩 (microcompact)        │
  │     ├── Snip (历史剪裁)              │
  │     ├── 上下文折叠 (context collapse) │
  │     └── 自动压缩 (autocompact)       │
  ├─────────────────────────────────────┤
  │  2. 流式 API 调用                   │
  │     ├── callModel()                 │
  │     ├── 处理 streaming fallback     │
  │     ├── 收集 tool_use blocks        │
  │     └── 流式工具执行（可选）          │
  ├─────────────────────────────────────┤
  │  3. 错误恢复                        │
  │     ├── prompt-too-long → 压缩重试  │
  │     ├── max_output_tokens → 恢复    │
  │     └── model fallback → 切模型重试 │
  ├─────────────────────────────────────┤
  │  4. 工具执行                        │
  │     ├── StreamingToolExecutor       │
  │     └── runTools()                  │
  ├─────────────────────────────────────┤
  │  5. 后处理                          │
  │     ├── Stop hooks                  │
  │     ├── Token budget 检查           │
  │     ├── 附件注入（记忆、队列命令）    │
  │     └── 判断：继续 or 结束           │
  └─────────────────────────────────────┘
}
```

### 流式 API 调用的实现

核心调用在 `query.ts:336`，通过 `deps.callModel()` 进行：

```typescript
for await (const message of deps.callModel({
  messages: prependUserContext(messagesForQuery, userContext),
  systemPrompt: fullSystemPrompt,
  thinkingConfig,
  tools,
  signal: toolUseContext.abortController.signal,
  options: {
    model: currentModel,
    toolChoice: undefined,
    isNonInteractiveSession,
    fallbackModel,
    onStreamingFallback: () => { streamingFallbackOccured = true },
    querySource,
    maxOutputTokensOverride,
    // ...
  },
})) {
  // 处理每条流式消息
}
```

每条流式消息的处理逻辑：

1. **`assistant` 消息**（`query.ts:378`）：
   - 检查 `tool_use` blocks → 如果有，标记 `needsFollowUp = true`
   - 如果启用了 `StreamingToolExecutor`，立即将 tool block 添加到执行器
   - yield 消息给调用方（实时渲染）

2. **`stream_event` 消息**（`QueryEngine.ts:479`）：
   - `message_start` → 重置当前消息用量
   - `message_delta` → 累加用量、捕获 `stop_reason`
   - `message_stop` → 累计总用量

### tool call 的解析和执行决策

工具调用的解析发生在 `query.ts:383`：

```typescript
const msgToolUseBlocks = message.message.content.filter(
  content => content.type === 'tool_use',
) as ToolUseBlock[]
if (msgToolUseBlocks.length > 0) {
  toolUseBlocks.push(...msgToolUseBlocks)
  needsFollowUp = true
}
```

**关键决策**：`needsFollowUp` 是整个循环的退出信号。如果 API 响应中没有 `tool_use` block，循环结束（`!needsFollowUp` → 走停止路径）。

### 流式工具执行（StreamingToolExecutor）

Claude Code 支持两种工具执行模式（`query.ts:266-270`）：

```typescript
let streamingToolExecutor = useStreamingToolExecution
  ? new StreamingToolExecutor(tools, canUseTool, toolUseContext)
  : null
```

**流式执行**：在 API 仍在返回内容时就开始执行已到达的 tool call。这大大减少了延迟——不需要等整个 API 响应完成。

```typescript
// query.ts:397 — 流式执行器中添加工具
if (streamingToolExecutor && !toolUseContext.abortController.signal.aborted) {
  for (const toolBlock of msgToolUseBlocks) {
    streamingToolExecutor.addTool(toolBlock, message)
  }
}
// query.ts:402 — 获取已完成的结果
for (const result of streamingToolExecutor.getCompletedResults()) {
  if (result.message) {
    yield result.message
    toolResults.push(result.message)
  }
}
```

**非流式执行**：等 API 完全返回后，调用 `runTools()` 批量执行：

```typescript
const toolUpdates = streamingToolExecutor
  ? streamingToolExecutor.getRemainingResults()
  : runTools(toolUseBlocks, assistantMessages, canUseTool, toolUseContext)
```

## 四、权限检查流程

权限检查在工具执行阶段发生。`canUseTool` 是一个注入的函数（`QueryEngine.ts:130`），每种权限模式（default/bypass/auto/plan）有自己的实现。

```
工具调用请求
  │
  ├── validateInput()           ← 工具自己的输入验证
  │
  ├── checkPermissions()        ← 工具特定的权限逻辑
  │
  ├── 权限模式检查
  │   ├── alwaysAllowRules      ← 命中 → allow
  │   ├── alwaysDenyRules       ← 命中 → deny
  │   └── 弹出权限对话框
  │       ├── allow-once        ← 本次通过，下次还问
  │       ├── allow-always      ← 持久化规则
  │       └── deny              ← 拒绝
  │
  └── 结果追踪
      └── permissionDenials[]   ← 记录所有拒绝
```

`Tool` 类型的 `checkPermissions()` 方法（`Tool.ts:422-429`）是工具级别的权限检查。权限上下文 `ToolPermissionContext`（`Tool.ts:123-139`）定义了：

```typescript
export type ToolPermissionContext = {
  mode: PermissionMode              // 'default' | 'bypass' | 'auto' | 'plan'
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  alwaysAllowRules: ToolPermissionRulesBySource
  alwaysDenyRules: ToolPermissionRulesBySource
  alwaysAskRules: ToolPermissionRulesBySource
  isBypassPermissionsModeAvailable: boolean
  shouldAvoidPermissionPrompts?: boolean  // 后台 agent 自动拒绝
}
```

## 五、重试和错误恢复机制

Claude Code 有多层错误恢复机制，每层针对不同类型的错误：

### 1. 模型降级（Fallback）

当 `callModel()` 抛出 `FallbackTriggeredError` 时（`query.ts:420`）：

```typescript
if (innerError instanceof FallbackTriggeredError && fallbackModel) {
  currentModel = fallbackModel
  attemptWithFallback = true
  // 清空已有结果，用新模型重试
  assistantMessages.length = 0
  toolResults.length = 0
  toolUseBlocks.length = 0
  needsFollowUp = false
}
```

### 2. Prompt 过长恢复

当收到 `413 prompt_too_long` 错误时（`query.ts:489-540`），有三级恢复：

```
prompt-too-long (413)
  │
  ├── 1. 上下文折叠排空 (collapse drain)
  │      └── 最便宜，保留细粒度上下文
  │
  ├── 2. 反应式压缩 (reactive compact)
  │      └── 全量摘要，单次尝试
  │
  └── 3. 抛出错误
```

### 3. 最大输出 Token 恢复

当模型输出被截断时（`query.ts:563-600`）：

```typescript
if (maxOutputTokensRecoveryCount < MAX_OUTPUT_TOKENS_RECOVERY_LIMIT) {
  // 注入恢复消息，让模型继续
  const recoveryMessage = createUserMessage({
    content: 'Output token limit hit. Resume directly...',
    isMeta: true,
  })
  // 继续下一轮迭代
}
```

最多重试 3 次（`MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3`）。

### 4. 流式降级（Streaming Fallback）

当流式 API 返回部分结果后失败时（`query.ts:362-376`）：

```typescript
if (streamingFallbackOccured) {
  // 为已产出的 assistant 消息创建 tombstone（墓碑）
  for (const msg of assistantMessages) {
    yield { type: 'tombstone', message: msg }
  }
  // 清空状态，重新创建流式执行器
  assistantMessages.length = 0
  streamingToolExecutor.discard()
  streamingToolExecutor = new StreamingToolExecutor(...)
}
```

Tombstone 消息通知 UI 移除已渲染的内容（这些中间结果可能有无效的 thinking 签名）。

## 六、Context Window 管理

Claude Code 采用了**多层压缩策略**，在不同阶段处理上下文溢出：

### 压缩层级

```
每轮迭代开始：
  │
  ├── 1. 微压缩 (microcompact)     ← 最轻量，工具结果截断
  │      query.ts:280-283
  │
  ├── 2. Snip (历史剪裁)            ← 移除远古消息，保留签名块
  │      query.ts:288-296
  │
  ├── 3. 上下文折叠 (context collapse) ← 细粒度折叠
  │      query.ts:304-313
  │
  └── 4. 自动压缩 (autocompact)     ← 最重量，全量摘要
         query.ts:318-325
```

### 自动压缩的触发条件

`autocompact` 检查在 `query.ts:318` 执行。它计算当前 token 数（减去 snip 释放的量），如果超过阈值则触发压缩：

```typescript
const { compactionResult, consecutiveFailures } = await deps.autocompact(
  messagesForQuery, toolUseContext,
  { systemPrompt, userContext, systemContext, toolUseContext,
    forkContextMessages: messagesForQuery },
  querySource, tracking, snipTokensFreed,
)
```

压缩后的处理（`query.ts:337-360`）：
- 生成 `compact_boundary` 系统消息
- 释放压缩前的消息（`splice(0, boundaryIdx)`）
- 重置 tracking 状态

### Token Budget 继续

当启用 `TOKEN_BUDGET` feature flag 时（`query.ts:618-642`），如果用户指定了 token 预算（如 `+500k`），系统会自动"推动"模型继续工作：

```typescript
const decision = checkTokenBudget(
  budgetTracker, agentId,
  getCurrentTurnTokenBudget(), getTurnOutputTokens(),
)
if (decision.action === 'continue') {
  // 注入 nudge message，继续下一轮
  state = {
    messages: [...messagesForQuery, ...assistantMessages,
      createUserMessage({ content: decision.nudgeMessage, isMeta: true })],
    // ...
  }
  continue
}
```

`checkTokenBudget()`（`tokenBudget.ts:28`）的逻辑：
- 使用量 < 90% 阈值 → 继续（注入 nudge message）
- 连续 3 轮增量 < 500 token → 收益递减，停止
- 否则 → 停止

## 七、State 对象：循环的状态传递

`query()` 内部的 while 循环使用一个 `State` 对象（`query.ts:105-117`）在迭代间传递状态：

```typescript
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number
  hasAttemptedReactiveCompact: boolean
  maxOutputTokensOverride: number | undefined
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined
  stopHookActive: boolean | undefined
  turnCount: number
  transition: Continue | undefined  // 上一轮为什么继续
}
```

每个"继续"路径都构造一个新的 `State`，记录 `transition.reason`：

```typescript
transition: { reason: 'next_turn' }                   // 正常下一轮
transition: { reason: 'reactive_compact_retry' }      // 压缩后重试
transition: { reason: 'max_output_tokens_recovery' }  // 输出截断恢复
transition: { reason: 'stop_hook_blocking' }           // Stop hook 阻塞
transition: { reason: 'token_budget_continuation' }   // Token 预算继续
transition: { reason: 'collapse_drain_retry' }         // 上下文折叠排空
```

这个 `transition` 字段主要用于测试断言——它让测试可以在不检查消息内容的情况下验证恢复路径是否触发。

## 总结

Claude Code 的 Tool-Use 循环是一个精心设计的**异步状态机**：

1. **异步生成器**架构让消息可以实时流式产出
2. **流式工具执行**大幅减少了 API 延迟和工具执行延迟的叠加
3. **多层错误恢复**确保了在 prompt 过长、输出截断、API 错误等场景下的鲁棒性
4. **四级压缩策略**在不同粒度上管理上下文窗口
5. **State 对象 + transition reason** 让循环的每个继续路径都是可追踪、可测试的

这种设计的哲学是：**永远不要让用户看到一个不可恢复的错误**。每一种失败都有对应的恢复路径，每一种恢复路径都有明确的边界条件。
