---
name: query-engine
description: "Understand and build AI agent execution engines that turn prompts into behavior. Use when (1) designing the tool-use loop for an agent system, (2) implementing prompt caching strategies for cost optimization, (3) building the prompt assembly pipeline (system prompt + tool prompts + memory + dynamic context), (4) debugging why an agent's behavior doesn't match its prompt, (5) optimizing agent latency and token usage. Triggers on phrases like agent execution engine, tool-use loop, prompt caching, prompt assembly, streaming API, context management, agent architecture."
---

# Query Engine

Prompt 不是静态字符串，是动态组装的分段系统。

## Core Architecture

```
用户输入 → QueryEngine.ask()
  ├─ 构建 System Prompt（7段静态 + N段动态）
  ├─ 注入 Context（CLAUDE.md + Git 状态）
  ├─ 加载 Memory（跨会话持久化）
  ├─ 组装 Tool Schemas
  └─ 调用 API（流式响应）
      └─ tool-use 循环 → 执行工具 → 返回结果 → 继续
```

## Step 1: Build the Tool-Use Loop

Agent 的核心是一个状态机：

```
发送消息 → 接收流式响应 → 解析 tool call
  ├─ 有 tool call → 权限检查 → 执行 → 注入结果 → 循环
  ├─ 无 tool call → 输出文本 → 结束
  └─ 错误 → 重试/降级/报告
```

关键设计决策：
- 权限检查在 tool call 解析后、执行前
- 流式响应中增量解析 tool call（不等完整响应）
- 四级错误恢复（模型降级、prompt 过长、输出截断、流式降级）

详见 [references/tool-use-loop.md](references/tool-use-loop.md)

## Step 2: Optimize Prompt Caching

Claude Code 的缓存策略核心：**静态在前，动态在后**。

```
[①-⑦ 静态段] ← scope: 'global'（跨用户缓存）
═══ __SYSTEM_PROMPT_DYNAMIC_BOUNDARY__ ═══
[⑧-⑮ 动态段] ← 每会话可能不同
```

两种缓存模式：
- `systemPromptSection()` — 计算一次，缓存直到 /clear
- `DANGEROUS_uncachedSystemPromptSection()` — 每回合重算，破坏缓存

Agent 列表分离优化：MCP 连接变化不破坏 tools-block 缓存。

详见 [references/prompt-caching.md](references/prompt-caching.md)

## Step 3: Assemble the Full Pipeline

从 prompt 到行为的完整链路：

```
getSystemPrompt() → 段缓存
fetchSystemPromptParts() → 并行获取 userContext + systemContext
assemble → 静态prompt + 动态prompt + memory + toolSchemas
注入 → system-reminder 增量追加
输出 → API 调用
```

system-reminder 标签系统：在对话过程中动态追加信息，不破坏已有缓存。

详见 [references/from-prompt-to-behavior.md](references/from-prompt-to-behavior.md)

## Key Rules

- **静态/动态分离是第一公民** — 缓存命中率直接决定成本和延迟
- **段级缓存 > 全量重算** — git status、CLAUDE.md 等不需要每回合重读
- **DANGEROUS_ 前缀是有意的** — 破坏缓存的段必须有显式理由
- **system-reminder 是增量注入机制** — 不同于 system prompt 的一次性注入
- **Feature flag 构建时死代码消除** — 运行时零开销
