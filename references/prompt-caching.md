# Prompt 缓存的工程优化

> 基于 `src/constants/prompts.ts`、`src/constants/systemPromptSections.ts` 源码分析

## 概述

Claude API 的 prompt caching 是按前缀匹配的——如果请求的 system prompt 前几个 block 和之前完全相同，中间结果可以复用。Claude Code 的 prompt 设计为此做了大量工程优化：静态/动态分离、段缓存、边界标记、Agent 列表排序……每一个细节都直接影响你的 API 账单。

## 一、缓存的基本原理

Claude API 的 prompt caching 基于**前缀匹配**：

```
请求 1:  [block_0][block_1][block_2][block_3][block_4]  → 写入缓存
请求 2:  [block_0][block_1][block_2][block_3'][block_4] → 命中前 3 个 block
                                                        → 只需计算 block_3' 和 block_4
```

**缓存命中的条件**：前缀必须完全一致（字节级别）。任何中间 block 的变化都会导致**后续所有 block 的缓存失效**。

这就是为什么 Claude Code 把 prompt 分为"静态"和"动态"两部分：

```
┌─────────────────────────────────┐  ← 静态：跨用户、跨会话不变
│  身份 + 安全指令                  │
│  行为规则                        │
│  任务执行指南                    │
│  风险操作确认                    │
│  工具偏好                        │
│  语气风格                        │
│  输出效率指令                    │
├═══════════════════════════════════┤  ← DYNAMIC BOUNDARY MARKER
│  session_guidance (会话指导)      │  ← 动态：每会话可能不同
│  memory (持久化记忆)             │
│  env_info (环境信息)             │
│  language (语言偏好)             │
│  output_style (输出风格)         │
│  mcp_instructions (MCP 指令)     │  ← ⚠️ 每次连接都可能变化
│  ...                            │
└─────────────────────────────────┘
```

## 二、静态/动态分离的完整实现

### 边界标记

`prompts.ts:68-74` 定义了边界标记：

```typescript
/**
 * Boundary marker separating static (cross-org cacheable) content
 * from dynamic content. Everything BEFORE this marker can use
 * scope: 'global'. Everything AFTER contains user/session-specific
 * content and should not be cached.
 */
export const SYSTEM_PROMPT_DYNAMIC_BOUNDARY =
  '__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__'
```

在 `getSystemPrompt()` 函数中（`prompts.ts:~670`），这个标记被插入到静态和动态内容之间：

```typescript
return [
  // --- Static content (cacheable) ---
  getSimpleIntroSection(outputStyleConfig),      // 身份 + 安全
  getSimpleSystemSection(),                       // 系统规则
  getSimpleDoingTasksSection(),                   // 任务指南
  getActionsSection(),                            // 风险操作
  getUsingYourToolsSection(enabledTools),         // 工具偏好
  getSimpleToneAndStyleSection(),                 // 语气风格
  getOutputEfficiencySection(),                   // 输出效率
  // === BOUNDARY MARKER - DO NOT MOVE OR REMOVE ===
  ...(shouldUseGlobalCacheScope()
    ? [SYSTEM_PROMPT_DYNAMIC_BOUNDARY]
    : []),
  // --- Dynamic content (registry-managed) ---
  ...resolvedDynamicSections,
].filter(s => s !== null)
```

**`shouldUseGlobalCacheScope()`** 控制是否插入边界标记。当 API beta header 支持 `cacheScope: 'global'` 时，边界前的内容可以跨用户缓存——这意味着数百万用户共享同一份静态 prompt 的缓存。

### 静态段详解

七段静态 prompt 的内容和设计意图：

| 段 | 函数 | 缓存稳定性 | 核心指令 |
|---|---|---|---|
| Intro | `getSimpleIntroSection()` | ⭐⭐⭐ | 身份定义、安全约束、URL 限制 |
| System | `getSimpleSystemSection()` | ⭐⭐⭐ | tool-result 标签、hooks、无限上下文 |
| Doing Tasks | `getSimpleDoingTasksSection()` | ⭐⭐⭐ (Ant 内置模型除外) | 不过度设计、反虚报、OWASP |
| Actions | `getActionsSection()` | ⭐⭐⭐ | 可逆/不可逆操作、审批范围 |
| Using Tools | `getUsingYourToolsSection()` | ⭐⭐ | 工具偏好链、并行调用 |
| Tone & Style | `getSimpleToneAndStyleSection()` | ⭐⭐ | emoji 限制、file:line 格式 |
| Output Efficiency | `getOutputEfficiencySection()` | ⭐⭐ | 直奔主题、不废话 |

**注意**：`getUsingYourToolsSection()` 和 `getSimpleDoingTasksSection()` 会根据 `USER_TYPE === 'ant'` 条件编译不同版本，这会影响缓存——但因为是构建时确定的，同一构建的所有用户仍然共享缓存。

## 三、systemPromptSection() vs DANGEROUS_uncachedSystemPromptSection()

`systemPromptSections.ts` 提供了两种段注册方式：

```typescript
// systemPromptSections.ts:15-20 — 缓存安全段
export function systemPromptSection(
  name: string, compute: ComputeFn,
): SystemPromptSection {
  return { name, compute, cacheBreak: false }
}

// systemPromptSections.ts:26-33 — 缓存破坏段
export function DANGEROUS_uncachedSystemPromptSection(
  name: string, compute: ComputeFn, _reason: string,
): SystemPromptSection {
  return { name, compute, cacheBreak: true }
}
```

### 段解析逻辑

```typescript
// systemPromptSections.ts:40-52
export async function resolveSystemPromptSections(
  sections: SystemPromptSection[],
): Promise<(string | null)[]> {
  const cache = getSystemPromptSectionCache()
  return Promise.all(
    sections.map(async s => {
      if (!s.cacheBreak && cache.has(s.name)) {
        return cache.get(s.name) ?? null  // 命中内存缓存
      }
      const value = await s.compute()
      setSystemPromptSectionCacheEntry(s.name, value)
      return value
    }),
  )
}
```

**`systemPromptSection`（`cacheBreak: false`）**：
- 第一次调用时 `compute()`，结果存入内存缓存
- 后续轮次直接返回缓存值
- **不会破坏 API prompt cache**——因为值不变，前缀匹配继续有效
- 适用场景：session_guidance、memory、language、output_style

**`DANGEROUS_uncachedSystemPromptSection`（`cacheBreak: true`）**：
- 每轮都重新 `compute()`
- 如果值变了，**后续所有 block 的 API 缓存全部失效**
- 命名 `DANGEROUS_` 是有意的——使用前需要写明理由
- 适用场景：MCP 指令（MCP 服务器随时可能连接/断开）

```typescript
// prompts.ts:~590 — MCP 指令是唯一的缓存破坏段
DANGEROUS_uncachedSystemPromptSection(
  'mcp_instructions',
  () => isMcpInstructionsDeltaEnabled()
    ? null
    : getMcpInstructionsSection(mcpClients),
  'MCP servers connect/disconnect between turns',
)
```

### 缓存清空

当用户执行 `/clear` 或 `/compact` 时（`systemPromptSections.ts:58-63`）：

```typescript
export function clearSystemPromptSections(): void {
  clearSystemPromptSectionState()    // 清空段缓存
  clearBetaHeaderLatches()           // 重置 beta header 评估
}
```

这意味着下一轮会重新计算所有段，但只要计算结果和之前相同，API 缓存仍然有效。

## 四、缓存命中/失效的触发条件

### 保持缓存命中的条件

1. **静态 prompt 不变** — 任何编辑 `getSimpleXxxSection()` 的修改都会破坏缓存
2. **工具列表不变** — 工具的 prompt 描述在 system prompt 中
3. **工具排序稳定** — `assembleToolPool()` 对工具名排序（见下文）
4. **动态段值不变** — session_guidance 等段的 `compute()` 返回值一致
5. **无 MCP 指令变化** — MCP 是唯一的 `cacheBreak` 段

### 缓存失效的常见场景

```
场景                          影响范围         缓存开销
─────────────────────────────────────────────────────
用户执行 /clear               重算段，但值通常不变   ~0
MCP 服务器连接/断开            mcp_instructions 变化  ~2000 token 重算
工具列表变化                  整个 prompt 前缀变化   全量重写
用户修改 CLAUDE.md            memory 段变化          ~500 token
切换语言偏好                  language 段变化         ~100 token
启用新 feature flag           相关段变化             视 flag 而定
```

## 五、Agent 列表分离的缓存优化

### 工具列表的排序策略

`tools.ts:298-316` 中的 `assembleToolPool()` 函数做了关键的排序优化：

```typescript
export function assembleToolPool(
  permissionContext: ToolPermissionContext,
  mcpTools: Tools,
): Tools {
  const builtInTools = getTools(permissionContext)
  const allowedMcpTools = filterToolsByDenyRules(mcpTools, permissionContext)

  // Sort each partition for prompt-cache stability
  const byName = (a: Tool, b: Tool) => a.name.localeCompare(b.name)
  return uniqBy(
    [...builtInTools].sort(byName).concat(allowedMcpTools.sort(byName)),
    'name',
  )
}
```

**关键设计**：内置工具和 MCP 工具**分别排序**，内置工具始终在前。

为什么要分两组排序？如果 flat sort（所有工具混合排序），每次 MCP 服务器连接时，新的 MCP 工具可能会排到内置工具之间，导致内置工具的 prompt 位置变化 → 静态 prompt 部分的缓存失效。

```
错误做法（flat sort）：
  [Agent, Bash, mcp__db__query, Edit, FileRead, ...]
  → MCP 工具 "mcp__db__query" 插入到 Bash 和 Edit 之间
  → Edit 及之后所有工具的 prompt 位置后移
  → 后续所有 block 缓存失效

正确做法（partition sort）：
  [Agent, Bash, Edit, FileRead, ... | mcp__db__query, mcp__git__status]
  → 内置工具排序固定
  → MCP 工具在后面，MCP 连接/断开只影响尾部
  → 内置工具部分的缓存完全保留
```

### 为什么 MCP 连接会破坏缓存

MCP 指令是通过 `DANGEROUS_uncachedSystemPromptSection` 注册的（`prompts.ts:~590`）。这是因为：

1. MCP 服务器可能在任何时刻连接或断开
2. MCP 指令包含服务器特定的行为指导（"使用 X 工具时注意 Y"）
3. 如果缓存旧的 MCP 指令，新连接的服务器不会被模型知道

新版 Claude Code 引入了 `mcp_instructions_delta` 机制——MCP 指令通过 attachment 注入（而非 system prompt），从而避免破坏 prompt 缓存：

```typescript
// 如果 delta 模式启用，MCP 指令走 attachment 路径
isMcpInstructionsDeltaEnabled()
  ? null  // 不注入到 system prompt
  : getMcpInstructionsSection(mcpClients)  // 传统路径（破坏缓存）
```

## 六、启动时并行预取的设计

`getSystemPrompt()` 函数在开头就启动了并行预取（`prompts.ts:~470`）：

```typescript
const cwd = getCwd()
const [skillToolCommands, outputStyleConfig, envInfo] = await Promise.all([
  getSkillToolCommands(cwd),        // 技能命令（文件系统读取）
  getOutputStyleConfig(),           // 输出风格配置
  computeSimpleEnvInfo(model, additionalWorkingDirectories),  // 环境信息
])
```

这里并行化了三个独立的 I/O 操作：
- `getSkillToolCommands()` — 需要扫描目录查找技能文件
- `getOutputStyleConfig()` — 读取配置文件
- `computeSimpleEnvInfo()` — 执行 `getIsGit()` 和 `getUnameSR()`

如果串行执行，总延迟是三者之和。并行执行后，延迟等于最慢的那个。

在 `query.ts` 中，记忆预取也采用了类似的策略（`query.ts:189`）：

```typescript
using pendingMemoryPrefetch = startRelevantMemoryPrefetch(
  state.messages, state.toolUseContext,
)
```

记忆预取在 API 流式调用期间后台执行（5-30秒），到工具执行完成后再消费结果。这把记忆加载的延迟完全隐藏在了 API 调用的时间窗口内。

## 七、$TMPDIR 替换的跨用户缓存优化

Claude Code 有一个不起眼但重要的优化：将用户特定的临时目录路径替换为通用占位符。

在构建 system prompt 时，某些路径（如 `/tmp/claude-u12345`）包含用户 ID，这会导致不同用户的 prompt 产生字节级差异，破坏跨用户的缓存。

通过替换为通用路径（如 `$TMPDIR/claude`），所有用户的 prompt 在这部分内容上完全一致，从而可以在 `cacheScope: 'global'` 级别共享缓存。

## 八、整体缓存策略架构

```
┌──────────────────────────────────────────────┐
│              cacheScope: 'global'             │
│  ┌──────────────────────────────────────────┐│
│  │  静态 prompt（7 段）                     ││  ← 跨所有用户共享
│  │  内置工具 prompt（排序后）                ││  ← 同一构建版本共享
│  └──────────────────────────────────────────┘│
├──────────────────────────────────────────────┤
│              缓存分界标记                      │
├──────────────────────────────────────────────┤
│              cacheScope: 'user'               │
│  ┌──────────────────────────────────────────┐│
│  │  session_guidance                        ││  ← 按会话变化
│  │  memory (CLAUDE.md)                      ││  ← 按项目变化
│  │  env_info                                ││  ← 按环境变化
│  │  language                                ││  ← 按用户变化
│  │  output_style                            ││  ← 按配置变化
│  │  mcp_instructions (cacheBreak)           ││  ← 随时变化 ⚠️
│  │  scratchpad, frc, summarize...           ││  ← 相对稳定
│  └──────────────────────────────────────────┘│
├──────────────────────────────────────────────┤
│  MCP 工具 prompt（排序后追加）                │  ← MCP 连接时变化
└──────────────────────────────────────────────┘
```

**缓存命中率估算**：

| 层级 | 典型命中率 | Token 节省 |
|---|---|---|
| 全局静态 | 95%+ | ~4000 token/请求 |
| 用户级动态 | 80-90% | ~2000 token/请求 |
| MCP 工具 | 70-80% | ~500-2000 token/请求 |

## 九、关键设计决策的工程权衡

### 1. 为什么不用更细粒度的缓存？

API 只支持前缀缓存，不支持"任意 block"缓存。这意味着：
- 不能跳过某个变化的 block 缓存后面的 block
- 所有动态内容必须放在静态内容之后
- 一个 `cacheBreak` 段会导致后续所有内容重算

### 2. 为什么 MCP 指令是唯一的 `cacheBreak` 段？

因为 MCP 是唯一一个**在对话过程中可能频繁变化**的输入。其他动态段（language、memory、env_info）在同一会话中通常不变。

但如果 `mcp_instructions_delta` feature 启用，MCP 指令走 attachment 路径（类似于文件变更通知），不进入 system prompt——从而消除了这个缓存破坏源。

### 3. 为什么工具排序这么重要？

在 system prompt 中，每个工具的描述是一个独立的文本块。如果工具 A 和工具 B 的位置互换，从工具 A 开始的所有后续工具的 prompt 内容都会"漂移"——即使内容相同，位置不同就意味着缓存前缀断裂。

通过**稳定排序**，确保在相同的工具集合下，prompt 的字节级表示始终一致。

## 总结

Claude Code 的 prompt 缓存优化是一系列精细的工程决策的集合：

1. **静态/动态分离**：将不变的内容放在前面，最大化跨请求的缓存命中
2. **段缓存机制**：`systemPromptSection` vs `DANGEROUS_uncached` 的二分法，精确控制每个段对缓存的影响
3. **工具列表分区排序**：内置工具和 MCP 工具分别排序，防止 MCP 连接破坏内置工具的缓存位置
4. **并行预取**：将 I/O 操作并行化，隐藏 prompt 组装的延迟
5. **Delta 注入**：MCP 指令走 attachment 路径，避免缓存破坏

这些优化的累积效果是：在典型使用场景下，Claude Code 的 prompt cache 命中率可以达到 90%+，每次请求节省数千 token 的计算和费用。
