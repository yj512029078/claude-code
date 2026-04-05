---
title: Prompt 管理系统
aliases: [Prompt 管理, Prompt Management]
series: Claude Code 架构解析
category: 核心运行时
order: 6
tags:
  - claude-code
  - prompt-engineering
  - system-prompt
  - dynamic-injection
  - context-management
date: 2026-04-04
---

# Prompt 管理系统 — 技术文档

## 目录

1. [系统概览](#1-系统概览)
2. [核心架构](#2-核心架构)
3. [System Prompt 构建管线](#3-system-prompt-构建管线)
4. [SystemPrompt 类型系统](#4-systemprompt-类型系统)
5. [Section 注册与缓存机制](#5-section-注册与缓存机制)
6. [Effective Prompt 优先级合并](#6-effective-prompt-优先级合并)
7. [User Context 与 System Context 注入](#7-user-context-与-system-context-注入)
8. [CLAUDE.md 记忆文件系统](#8-claudemd-记忆文件系统)
9. [Auto Memory (MEMORY.md) 系统](#9-auto-memory-memorymd-系统)
10. [Tool Prompt 描述系统](#10-tool-prompt-描述系统)
11. [API 层 Prompt 装配](#11-api-层-prompt-装配)
12. [Prompt Cache 优化策略](#12-prompt-cache-优化策略)
13. [多模式 Prompt 差异](#13-多模式-prompt-差异)
14. [Compact 摘要 Prompt](#14-compact-摘要-prompt)
15. [端到端数据流](#15-端到端数据流)
16. [关键文件索引](#16-关键文件索引)

---

## 1. 系统概览

Claude Code 的 Prompt 管理是一个 **多层组合式架构**，将系统提示的构建拆分为静态指令、动态 Section、用户上下文、系统上下文、工具描述等多个独立管道，最终在 API 调用前汇合成完整的请求负载。

核心设计目标：
- **可缓存性**：通过 `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 分界标记将静态内容与动态内容分离，最大化 Anthropic API 的 prompt cache 命中率
- **可组合性**：Section 注册表模式允许各模块独立注册 prompt 片段，无需修改主构建函数
- **多模式适配**：Agent 模式、Plan 模式、Coordinator 模式、Proactive 模式各有不同的 prompt 组装路径
- **缓存稳定性**：工具 schema 会话级缓存、Section 结果缓存、memoize 上下文缓存，防止 GrowthBook feature flag 翻转导致缓存失效

整体 Prompt 分为三大通道注入 API 请求：

| 通道 | 载体 | 内容 |
|------|------|------|
| `system` 参数 | `TextBlockParam[]` | 身份前缀 + 静态指令 + 动态 Section + 系统上下文 |
| `messages[0]` (meta user msg) | `<system-reminder>` 包裹的文本 | CLAUDE.md 内容 + 当前日期 |
| `tools` 参数 | JSON Schema + description | 每个工具的描述文本（由 `tool.prompt()` 生成） |

---

## 2. 核心架构

```
┌────────────────────────────────────────────────────────────────┐
│                      API Request 最终形态                        │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ system: TextBlockParam[]                                │   │
│  │  ├─ [0] Attribution Header (无缓存)                     │   │
│  │  ├─ [1] CLI Identity Prefix (org 级缓存)                │   │
│  │  ├─ [2] Static Content (global 级缓存)                  │   │
│  │  │      ├─ Intro Section                                │   │
│  │  │      ├─ System Section                               │   │
│  │  │      ├─ Doing Tasks Section                          │   │
│  │  │      ├─ Actions Section                              │   │
│  │  │      ├─ Using Your Tools Section                     │   │
│  │  │      ├─ Tone & Style Section                         │   │
│  │  │      └─ Output Efficiency Section                    │   │
│  │  ├─ ─ ─ SYSTEM_PROMPT_DYNAMIC_BOUNDARY ─ ─ ─           │   │
│  │  └─ [3] Dynamic Content (无缓存)                        │   │
│  │         ├─ Session Guidance                             │   │
│  │         ├─ Memory (MEMORY.md)                           │   │
│  │         ├─ Environment Info                             │   │
│  │         ├─ Language Preference                          │   │
│  │         ├─ Output Style                                 │   │
│  │         ├─ MCP Instructions                             │   │
│  │         ├─ Scratchpad Instructions                      │   │
│  │         └─ Git Status / Cache Breaker                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ messages:                                               │   │
│  │  ├─ [0] meta user msg: <system-reminder>               │   │
│  │  │      ├─ claudeMd (CLAUDE.md 内容)                    │   │
│  │  │      └─ currentDate                                  │   │
│  │  ├─ [1..n] conversation history                         │   │
│  │  └─ [n+1] latest user message                           │   │
│  └─────────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ tools: BetaToolUnion[]                                  │   │
│  │  ├─ { name, description: tool.prompt(), input_schema }  │   │
│  │  └─ 最后一个工具可带 cache_control                        │   │
│  └─────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────┘
```

---

## 3. System Prompt 构建管线

### 3.1 入口函数 `getSystemPrompt()`

**文件**: `src/constants/prompts.ts`

```typescript
export async function getSystemPrompt(
  tools: Tools,
  model: string,
  additionalWorkingDirectories?: string[],
  mcpClients?: MCPServerConnection[],
): Promise<string[]>
```

这是主系统 prompt 的构建入口，返回 `string[]`（每个元素将成为 API 请求中 system 参数的一个文本块）。

**简化模式短路**：当 `CLAUDE_CODE_SIMPLE` 环境变量为真时，仅返回极简 prompt：

```typescript
if (isEnvTruthy(process.env.CLAUDE_CODE_SIMPLE)) {
  return [`You are Claude Code, Anthropic's official CLI for Claude.\n\nCWD: ${getCwd()}\nDate: ${getSessionStartDate()}`]
}
```

**Proactive 模式短路**：当处于 Proactive/KAIROS 模式时，使用精简的自主代理 prompt：

```typescript
return [
  `\nYou are an autonomous agent. Use the available tools to do useful work.\n\n${CYBER_RISK_INSTRUCTION}`,
  getSystemRemindersSection(),
  await loadMemoryPrompt(),
  envInfo,
  getLanguageSection(settings.language),
  getMcpInstructionsSection(mcpClients),
  getScratchpadInstructions(),
  getFunctionResultClearingSection(model),
  SUMMARIZE_TOOL_RESULTS_SECTION,
  getProactiveSection(),
].filter(s => s !== null)
```

### 3.2 标准路径 — 静态 + 动态双区

标准构建路径将 prompt 分为 **静态区** 和 **动态区**，以 `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 分隔：

```typescript
return [
  // === 静态内容（跨组织可缓存）===
  getSimpleIntroSection(outputStyleConfig),     // 身份与安全指令
  getSimpleSystemSection(),                     // 系统行为规则
  getSimpleDoingTasksSection(),                 // 任务执行指南
  getActionsSection(),                          // 操作安全约束
  getUsingYourToolsSection(enabledTools),       // 工具使用指南
  getSimpleToneAndStyleSection(),               // 语气风格
  getOutputEfficiencySection(),                 // 输出效率
  // === 分界标记 ===
  ...(shouldUseGlobalCacheScope() ? [SYSTEM_PROMPT_DYNAMIC_BOUNDARY] : []),
  // === 动态内容（注册表驱动）===
  ...resolvedDynamicSections,
].filter(s => s !== null)
```

### 3.3 静态 Section 详解

| Section 函数 | 职责 |
|---|---|
| `getSimpleIntroSection()` | 身份声明、网络安全风险指令（CYBER_RISK_INSTRUCTION）、URL 限制 |
| `getSimpleSystemSection()` | 工具权限模式说明、`<system-reminder>` 标签说明、hooks 交互、自动压缩说明 |
| `getSimpleDoingTasksSection()` | 代码风格约束（最小化改动、避免过度工程化、不添加多余注释）、安全编码、错误处理策略 |
| `getActionsSection()` | 操作安全——可逆性判断、高风险操作确认（如 force push、删除文件、发消息） |
| `getUsingYourToolsSection()` | 工具优先级（FileRead > cat、FileEdit > sed）、并行调用策略、任务管理工具 |
| `getSimpleToneAndStyleSection()` | 禁止 emoji、简洁输出、引用格式（file:line）、GitHub issue 格式 |
| `getOutputEfficiencySection()` | Ant 用户：逆金字塔写作风格；外部用户：简洁直接 |

### 3.4 动态 Section 注册表

动态 Section 通过 `systemPromptSection()` 和 `DANGEROUS_uncachedSystemPromptSection()` 注册：

```typescript
const dynamicSections = [
  systemPromptSection('session_guidance', () =>
    getSessionSpecificGuidanceSection(enabledTools, skillToolCommands),
  ),
  systemPromptSection('memory', () => loadMemoryPrompt()),
  systemPromptSection('env_info_simple', () =>
    computeSimpleEnvInfo(model, additionalWorkingDirectories),
  ),
  systemPromptSection('language', () => getLanguageSection(settings.language)),
  systemPromptSection('output_style', () => getOutputStyleSection(outputStyleConfig)),
  DANGEROUS_uncachedSystemPromptSection(
    'mcp_instructions',
    () => isMcpInstructionsDeltaEnabled() ? null : getMcpInstructionsSection(mcpClients),
    'MCP servers connect/disconnect between turns',
  ),
  systemPromptSection('scratchpad', () => getScratchpadInstructions()),
  systemPromptSection('frc', () => getFunctionResultClearingSection(model)),
  systemPromptSection('summarize_tool_results', () => SUMMARIZE_TOOL_RESULTS_SECTION),
  // Ant-only: 数值长度锚点
  // Feature-gated: token budget, brief
]
```

---

## 4. SystemPrompt 类型系统

**文件**: `src/utils/systemPromptType.ts`

```typescript
export type SystemPrompt = readonly string[] & {
  readonly __brand: 'SystemPrompt'
}

export function asSystemPrompt(value: readonly string[]): SystemPrompt {
  return value as SystemPrompt
}
```

使用 TypeScript **品牌类型（Branded Type）** 模式来区分普通字符串数组与已组装的系统 prompt，防止未经 `asSystemPrompt()` 包装的原始数组被传入 API 层。

该模块故意 **零依赖**，避免被高层模块引入时引发循环初始化。

---

## 5. Section 注册与缓存机制

**文件**: `src/constants/systemPromptSections.ts`

### 5.1 两种 Section 类型

**普通缓存 Section**：计算一次，缓存到 `/clear` 或 `/compact` 为止：

```typescript
export function systemPromptSection(
  name: string,
  compute: ComputeFn,
): SystemPromptSection {
  return { name, compute, cacheBreak: false }
}
```

**危险的非缓存 Section**：每轮重新计算，会打破 prompt cache：

```typescript
export function DANGEROUS_uncachedSystemPromptSection(
  name: string,
  compute: ComputeFn,
  _reason: string,  // 必须说明为什么需要打破缓存
): SystemPromptSection {
  return { name, compute, cacheBreak: true }
}
```

### 5.2 解析流程

```typescript
export async function resolveSystemPromptSections(
  sections: SystemPromptSection[],
): Promise<(string | null)[]> {
  const cache = getSystemPromptSectionCache()
  return Promise.all(
    sections.map(async s => {
      if (!s.cacheBreak && cache.has(s.name)) {
        return cache.get(s.name) ?? null    // 命中缓存
      }
      const value = await s.compute()        // 计算并缓存
      setSystemPromptSectionCacheEntry(s.name, value)
      return value
    }),
  )
}
```

### 5.3 缓存清除

`/clear` 和 `/compact` 命令会调用 `clearSystemPromptSections()`，同时重置 beta header latches：

```typescript
export function clearSystemPromptSections(): void {
  clearSystemPromptSectionState()
  clearBetaHeaderLatches()
}
```

---

## 6. Effective Prompt 优先级合并

**文件**: `src/utils/systemPrompt.ts`

`buildEffectiveSystemPrompt()` 实现多源 prompt 的优先级合并，决定最终使用哪个系统 prompt：

```typescript
export function buildEffectiveSystemPrompt({
  mainThreadAgentDefinition,
  toolUseContext,
  customSystemPrompt,
  defaultSystemPrompt,
  appendSystemPrompt,
  overrideSystemPrompt,
}): SystemPrompt
```

### 6.1 优先级（从高到低）

| 优先级 | 来源 | 行为 |
|--------|------|------|
| 0 | `overrideSystemPrompt` | 完全替代，忽略所有其他来源（如 loop mode） |
| 1 | Coordinator 模式 | 使用 `getCoordinatorSystemPrompt()` + 可选 `appendSystemPrompt` |
| 2 | Agent 定义 | `mainThreadAgentDefinition.getSystemPrompt()` |
| 2a | Agent + Proactive | Agent prompt **追加**到默认 prompt（而非替代） |
| 3 | `customSystemPrompt` | `--system-prompt` CLI 参数指定 |
| 4 | `defaultSystemPrompt` | 标准 `getSystemPrompt()` 输出 |
| +∞ | `appendSystemPrompt` | 始终追加到最终结果末尾（override 除外） |

### 6.2 关键逻辑

```typescript
// Override 最高优先
if (overrideSystemPrompt) {
  return asSystemPrompt([overrideSystemPrompt])
}

// Coordinator 模式
if (isCoordinatorMode && !mainThreadAgentDefinition) {
  return asSystemPrompt([getCoordinatorSystemPrompt(), ...append])
}

// Proactive 模式下 Agent prompt 追加到默认
if (agentSystemPrompt && isProactiveActive) {
  return asSystemPrompt([
    ...defaultSystemPrompt,
    `\n# Custom Agent Instructions\n${agentSystemPrompt}`,
    ...append,
  ])
}

// 标准路径：Agent > Custom > Default
return asSystemPrompt([
  ...(agentSystemPrompt ? [agentSystemPrompt]
    : customSystemPrompt ? [customSystemPrompt]
    : defaultSystemPrompt),
  ...append,
])
```

---

## 7. User Context 与 System Context 注入

**文件**: `src/context.ts`, `src/utils/api.ts`

### 7.1 User Context（用户上下文）

通过 `getUserContext()` 获取，包含 **CLAUDE.md 内容** 和 **当前日期**。使用 `lodash.memoize` 缓存整个会话周期：

```typescript
export const getUserContext = memoize(async (): Promise<{ [k: string]: string }> => {
  const claudeMd = shouldDisableClaudeMd ? null
    : getClaudeMds(filterInjectedMemoryFiles(await getMemoryFiles()))

  return {
    ...(claudeMd && { claudeMd }),
    currentDate: `Today's date is ${getLocalISODate()}.`,
  }
})
```

**注入方式**：不是放入 `system` 参数，而是通过 `prependUserContext()` 作为 **第一条 user message** 插入消息列表：

```typescript
export function prependUserContext(
  messages: Message[],
  context: { [k: string]: string },
): Message[] {
  return [
    createUserMessage({
      content: `<system-reminder>
As you answer the user's questions, you can use the following context:
# claudeMd
${claudeMdContent}
# currentDate
Today's date is 2026-04-04.

IMPORTANT: this context may or may not be relevant to your tasks...
</system-reminder>`,
      isMeta: true,
    }),
    ...messages,
  ]
}
```

设计理由：将 CLAUDE.md 放在 user message 中而非 system prompt 中，可以减少 system prompt 的变化频率，保护 prompt cache 命中率。

### 7.2 System Context（系统上下文）

通过 `getSystemContext()` 获取，包含 **Git 状态快照** 和可选的 **cache breaker**：

```typescript
export const getSystemContext = memoize(async () => {
  const gitStatus = await getGitStatus()  // branch, status, recent commits, user name
  const injection = getSystemPromptInjection()  // ant-only cache breaking

  return {
    ...(gitStatus && { gitStatus }),
    ...(injection ? { cacheBreaker: `[CACHE_BREAKER: ${injection}]` } : {}),
  }
})
```

**注入方式**：通过 `appendSystemContext()` 追加到 system prompt 数组末尾：

```typescript
export function appendSystemContext(
  systemPrompt: SystemPrompt,
  context: { [k: string]: string },
): string[] {
  return [
    ...systemPrompt,
    Object.entries(context)
      .map(([key, value]) => `${key}: ${value}`)
      .join('\n'),
  ].filter(Boolean)
}
```

### 7.3 Git Status 详情

```typescript
export const getGitStatus = memoize(async () => {
  const [branch, mainBranch, status, log, userName] = await Promise.all([
    getBranch(),
    getDefaultBranch(),
    execFileNoThrow(gitExe(), ['status', '--short']),
    execFileNoThrow(gitExe(), ['log', '--oneline', '-n', '5']),
    execFileNoThrow(gitExe(), ['config', 'user.name']),
  ])

  // 超过 2000 字符的 status 会被截断
  const truncatedStatus = status.length > MAX_STATUS_CHARS
    ? status.substring(0, MAX_STATUS_CHARS) + '\n... (truncated...)'
    : status

  return [
    'This is the git status at the start of the conversation...',
    `Current branch: ${branch}`,
    `Main branch: ${mainBranch}`,
    `Git user: ${userName}`,
    `Status:\n${truncatedStatus}`,
    `Recent commits:\n${log}`,
  ].join('\n\n')
})
```

---

## 8. CLAUDE.md 记忆文件系统

**文件**: `src/utils/claudemd.ts`

### 8.1 加载优先级

文件按以下顺序加载，**后加载的优先级更高**（模型对后出现的内容关注度更高）：

| 顺序 | 类型 | 路径示例 |
|------|------|----------|
| 1 | Managed Memory | `/etc/claude-code/CLAUDE.md` |
| 2 | User Memory | `~/.claude/CLAUDE.md` |
| 3 | Project Memory | 项目根目录的 `CLAUDE.md`, `.claude/CLAUDE.md`, `.claude/rules/*.md` |
| 4 | Local Memory | 项目根目录的 `CLAUDE.local.md` |

### 8.2 文件发现

- **User Memory**：从用户主目录加载
- **Project + Local**：从当前目录向上遍历到根目录，越靠近当前目录的文件优先级越高
- 每个目录检查：`CLAUDE.md`、`.claude/CLAUDE.md`、`.claude/rules/*.md`

### 8.3 @include 指令

CLAUDE.md 文件支持 `@` 引用其他文件：
- `@path` — 相对路径
- `@./relative/path` — 显式相对路径
- `@~/home/path` — 用户主目录路径
- `@/absolute/path` — 绝对路径

限制：仅在叶子文本节点中生效（不在代码块内），有循环引用检测。

### 8.4 注入 Prompt

CLAUDE.md 内容前会加上强调指令：

```typescript
const MEMORY_INSTRUCTION_PROMPT =
  'Codebase and user instructions are shown below. Be sure to adhere to these instructions. IMPORTANT: These instructions OVERRIDE any default behavior and you MUST follow them exactly as written.'
```

### 8.5 子代理优化

子代理（subagent）可选择省略 CLAUDE.md 以降低成本：

```typescript
// src/tools/AgentTool/runAgent.ts
const omitClaudeMd = getFeatureValue('tengu_slim_subagent_claudemd')
```

---

## 9. Auto Memory (MEMORY.md) 系统

**文件**: `src/memdir/memdir.ts`

自动记忆系统通过 `loadMemoryPrompt()` 将 MEMORY.md 内容作为 **系统 prompt section** 注入（不同于 CLAUDE.md 的 user context 通道）。

```typescript
systemPromptSection('memory', () => loadMemoryPrompt())
```

MEMORY.md 的约束：
- 最大行数：200 行（`MAX_ENTRYPOINT_LINES`）
- 最大字节数：25,000 字节（`MAX_ENTRYPOINT_BYTES`）
- 超限时自动截断并附加警告

---

## 10. Tool Prompt 描述系统

### 10.1 每个工具的 prompt

每个工具在 `src/tools/<ToolName>/prompt.ts` 中定义描述文本，通过 `tool.prompt()` 方法返回：

```typescript
// Tool 基类中的 prompt 方法签名 (src/Tool.ts)
prompt(options: {
  getToolPermissionContext: () => Promise<ToolPermissionContext>
  tools: Tools
  agents: AgentDefinition[]
  allowedAgentTypes?: string[]
}): Promise<string>
```

### 10.2 工具 Schema 构建

**文件**: `src/utils/api.ts`

`toolToAPISchema()` 将工具转换为 API 格式：

```typescript
export async function toolToAPISchema(tool: Tool, options): Promise<BetaToolUnion> {
  const base = {
    name: tool.name,
    description: await tool.prompt({...}),    // 工具描述（自然语言）
    input_schema: zodToJsonSchema(tool.inputSchema),  // 参数 Schema
  }

  // 可选：strict mode、eager_input_streaming、defer_loading
  // ...
}
```

### 10.3 工具 Schema 会话级缓存

**文件**: `src/utils/toolSchemaCache.ts`

为防止 GrowthBook flag 翻转或 `tool.prompt()` 漂移导致缓存失效，工具 schema 在会话内只计算一次：

```typescript
const cache = getToolSchemaCache()
let base = cache.get(cacheKey)
if (!base) {
  base = { name, description: await tool.prompt(...), input_schema }
  cache.set(cacheKey, base)
}
```

缓存键：普通工具用 `tool.name`，动态 schema 工具（如 StructuredOutput）用 `${tool.name}:${JSON.stringify(inputJSONSchema)}`。

### 10.4 全局工具使用指南

除了每个工具自身的描述，`getSystemPrompt()` 还在系统 prompt 中注入全局工具使用指南：

- `getUsingYourToolsSection()` — 工具优先级、并行调用策略
- `getSessionSpecificGuidanceSection()` — Agent 工具、Skill 工具、Verification Agent 指南
- `getAgentToolSection()` — 子代理使用策略

---

## 11. API 层 Prompt 装配

**文件**: `src/services/api/claude.ts`

### 11.1 身份前缀注入

在 `queryModel()` 中，系统 prompt 数组前面会注入身份前缀：

```typescript
systemPrompt = asSystemPrompt([
  getAttributionHeader(fingerprint),          // 计费归因头
  getCLISyspromptPrefix({                     // 身份前缀
    isNonInteractive,
    hasAppendSystemPrompt,
  }),
  ...systemPrompt,                            // 核心 prompt 内容
  ...(advisorModel ? [ADVISOR_TOOL_INSTRUCTIONS] : []),
  ...(injectChromeHere ? [CHROME_TOOL_SEARCH_INSTRUCTIONS] : []),
].filter(Boolean))
```

### 11.2 身份前缀变体

**文件**: `src/constants/system.ts`

```typescript
const DEFAULT_PREFIX = `You are Claude Code, Anthropic's official CLI for Claude.`
const AGENT_SDK_CLAUDE_CODE_PRESET_PREFIX = `You are Claude Code, Anthropic's official CLI for Claude, running within the Claude Agent SDK.`
const AGENT_SDK_PREFIX = `You are a Claude agent, built on Anthropic's Claude Agent SDK.`
```

选择逻辑：
- Vertex 提供商 → `DEFAULT_PREFIX`
- 非交互 + 有 appendSystemPrompt → `AGENT_SDK_CLAUDE_CODE_PRESET_PREFIX`
- 非交互 → `AGENT_SDK_PREFIX`
- 交互式 → `DEFAULT_PREFIX`

### 11.3 Attribution Header

```typescript
export function getAttributionHeader(fingerprint: string): string {
  const version = `${MACRO.VERSION}.${fingerprint}`
  const entrypoint = process.env.CLAUDE_CODE_ENTRYPOINT ?? 'unknown'
  return `x-anthropic-billing-header: cc_version=${version}; cc_entrypoint=${entrypoint};`
}
```

### 11.4 System Prompt 块构建

```typescript
export function buildSystemPromptBlocks(
  systemPrompt: SystemPrompt,
  enablePromptCaching: boolean,
  options?,
): TextBlockParam[] {
  return splitSysPromptPrefix(systemPrompt, options).map(block => ({
    type: 'text',
    text: block.text,
    ...(enablePromptCaching && block.cacheScope !== null && {
      cache_control: getCacheControl({ scope: block.cacheScope }),
    }),
  }))
}
```

---

## 12. Prompt Cache 优化策略

### 12.1 分界标记

```typescript
export const SYSTEM_PROMPT_DYNAMIC_BOUNDARY = '__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__'
```

位于 `getSystemPrompt()` 输出中，分隔静态内容（可 global 缓存）和动态内容（不缓存或 org 级缓存）。

### 12.2 `splitSysPromptPrefix()` 的三种模式

**文件**: `src/utils/api.ts`

| 模式 | 条件 | 块划分 |
|------|------|--------|
| MCP 工具存在 | `skipGlobalCacheForSystemPrompt=true` | Attribution(null) + Prefix(org) + Rest(org) |
| Global 缓存 + 有分界 | 1P 提供商，分界标记存在 | Attribution(null) + Prefix(null) + Static(global) + Dynamic(null) |
| 默认 | 3P 提供商或无分界 | Attribution(null) + Prefix(org) + Rest(org) |

### 12.3 缓存层级

```
┌──────────────────┐
│  global (跨组织) │  ← 静态指令（不含用户/会话特定内容）
├──────────────────┤
│  org (组织级)    │  ← 身份前缀 + 包含组织配置的内容
├──────────────────┤
│  null (不缓存)   │  ← Attribution header, 动态内容
└──────────────────┘
```

### 12.4 消息级缓存

`addCacheBreakpoints()` 在消息列表中选择一条消息的最后一个 content block 添加 `cache_control`。标记位置通常是倒数第一或倒数第二条消息。

### 12.5 工具级缓存

最后一个工具 schema 可以带 `cache_control`，使整个工具列表可被缓存。

### 12.6 Cache TTL

```typescript
function getCacheControl({ scope, querySource }): CacheControl {
  return {
    type: 'ephemeral',
    ...(scope && { scope }),
    ...(should1hCacheTTL() && { ttl: '1h' }),
  }
}
```

### 12.7 缓存稳定性保护

- **Beta header latches**：`afkHeaderLatched`、`fastModeHeaderLatched`、`cacheEditingHeaderLatched` 等一旦开启就保持到 `/clear`，避免 mid-session 翻转打破缓存
- **Tool schema cache**：每会话每工具只计算一次
- **Section cache**：`systemPromptSection` 注册的内容只计算一次
- **Context memoize**：`getUserContext`/`getSystemContext` 使用 lodash memoize
- **Cache break detection**：`promptCacheBreakDetection.ts` 哈希系统 prompt，当缓存读取 token 异常下降时诊断原因

---

## 13. 多模式 Prompt 差异

### 13.1 标准 Agent 模式

使用完整的 `getSystemPrompt()` + `buildEffectiveSystemPrompt()`，走标准管线。

### 13.2 Plan 模式

Plan 模式的指令不是通过系统 prompt 注入，而是通过 **消息附件（attachments）** 作为 `<system-reminder>` 包裹的 user meta message 注入：

```typescript
// src/utils/messages.ts 中的 normalizeAttachmentForAPI
// plan_mode attachment → 完整 plan mode 指令
// plan_mode_sparse → 精简提醒
// plan_mode_exit → 退出指令
```

Plan 模式还影响模型选择：当 `permissionMode === 'plan'` 且上下文超过 200k token 时，会切换到不同的运行时模型。

### 13.3 Coordinator 模式

```typescript
if (isCoordinatorMode && !mainThreadAgentDefinition) {
  const { getCoordinatorSystemPrompt } = require('../coordinator/coordinatorMode.js')
  return asSystemPrompt([getCoordinatorSystemPrompt(), ...append])
}
```

完全替换默认 prompt，使用协调者专用 prompt。同时 `getCoordinatorUserContext` 会合并到 user context 中。

### 13.4 Proactive/KAIROS 模式

使用极简自主代理 prompt + Memory + Proactive Section，Agent 定义的 prompt 追加（而非替换）到默认 prompt。

### 13.5 子代理/Teammate 模式

- `src/tools/AgentTool/runAgent.ts` — 子代理运行时构建独立的系统 prompt
- `src/utils/swarm/teammatePromptAddendum.ts` — Teammate 追加 prompt
- `systemPromptMode` 控制行为：`default`（使用默认）、`replace`（完全替换）、`append`（追加到默认）

### 13.6 内置 Agent 类型

各在 `src/tools/AgentTool/built-in/` 中定义独立 prompt：

| Agent | 文件 | 用途 |
|-------|------|------|
| Verification Agent | `verificationAgent.ts` | 独立对抗性验证 |
| Explore Agent | `exploreAgent.ts` | 代码库探索 |
| Plan Agent | `planAgent.ts` | 规划 |
| General Purpose | `generalPurposeAgent.ts` | 通用研究 |

---

## 14. Compact 摘要 Prompt

**文件**: `src/services/compact/prompt.ts`

当对话接近上下文窗口限制时，自动触发 compact（摘要压缩）。

### 14.1 Compact Prompt 结构

```
CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.
[NO_TOOLS_PREAMBLE — 禁止工具调用的强调指令]

Your task is to create a detailed summary of the conversation...
[DETAILED_ANALYSIS_INSTRUCTION — 分析步骤]

Your summary should include:
1. Primary Request and Intent
2. Key Technical Concepts
3. Files and Code Sections
4. Errors and fixes
5. Problem Solving
6. All user messages
7. Pending Tasks
8. Current Work
9. Optional Next Step

REMINDER: Do NOT call any tools...
[NO_TOOLS_TRAILER — 再次强调]
```

### 14.2 三种 Compact 变体

| 变体 | 函数 | 场景 |
|------|------|------|
| 全量 Compact | `getCompactPrompt()` | 压缩整个对话 |
| 部分 Compact (from) | `getPartialCompactPrompt('from')` | 仅压缩最近的消息 |
| 部分 Compact (up_to) | `getPartialCompactPrompt('up_to')` | 压缩前半部分，保留后续消息 |

### 14.3 摘要后处理

`formatCompactSummary()` 移除 `<analysis>` 草稿块（仅用于提升摘要质量的思维链），保留 `<summary>` 内容。

### 14.4 自动 Compact 触发

- `tokenCountWithEstimation()` 估算当前上下文 token 数
- `getAutoCompactThreshold()` = `effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS (13,000)`
- 当估算 token 数超过阈值时触发 `autoCompactIfNeeded()`

---

## 15. 端到端数据流

```
用户输入文本/图片
    │
    ▼
processTextPrompt() / processUserInput()
    │  ├─ 处理斜杠命令
    │  ├─ 处理附件
    │  └─ 创建 UserMessage
    │
    ▼
handlePromptSubmit() → 加入消息队列
    │
    ▼
query() 主循环
    │
    ├──[1] 消息准备
    │   ├─ getMessagesAfterCompactBoundary() — 截取 compact 边界后的消息
    │   ├─ snipCompactIfNeeded() — 可选 snip 旧历史
    │   ├─ applyToolResultBudget() — 工具结果预算
    │   └─ autoCompactIfNeeded() — 自动 compact
    │
    ├──[2] Prompt 组装
    │   ├─ fullSystemPrompt = appendSystemContext(systemPrompt, systemContext)
    │   └─ messagesForQuery = prependUserContext(messagesForQuery, userContext)
    │
    ├──[3] deps.callModel() → queryModelWithStreaming()
    │   │
    │   ▼
    │   queryModel() (src/services/api/claude.ts)
    │   │
    │   ├──[3a] normalizeMessagesForAPI() — 消息标准化
    │   ├──[3b] ensureToolResultPairing() — 工具结果配对
    │   ├──[3c] stripExcessMediaItems() — 媒体数量限制
    │   ├──[3d] 身份前缀注入
    │   │   systemPrompt = [attribution, cliPrefix, ...systemPrompt, ...]
    │   ├──[3e] buildSystemPromptBlocks() — 带 cache_control 的文本块
    │   ├──[3f] toolToAPISchema() — 工具描述 + schema
    │   ├──[3g] addCacheBreakpoints() — 消息级缓存标记
    │   │
    │   └──[3h] anthropic.beta.messages.create({ system, messages, tools, stream: true })
    │
    ├──[4] 流式响应处理
    │   ├─ Assistant messages 收集
    │   └─ Tool use blocks 检测
    │
    ├──[5] 工具执行
    │   ├─ StreamingToolExecutor.runToolUse()
    │   └─ 工具结果 → UserMessage (tool_result blocks)
    │
    └──[6] 递归下一轮
        messages = [...prior, ...assistantMessages, ...toolResults]
        → 回到 [1]
```

---

## 16. 关键文件索引

### 核心 Prompt 构建

| 文件 | 职责 |
|------|------|
| `src/constants/prompts.ts` | 主系统 prompt 构建入口 `getSystemPrompt()`，所有静态和动态 Section 定义 |
| `src/constants/systemPromptSections.ts` | Section 注册表：`systemPromptSection()`、`DANGEROUS_uncachedSystemPromptSection()`、`resolveSystemPromptSections()` |
| `src/utils/systemPrompt.ts` | 优先级合并：`buildEffectiveSystemPrompt()` |
| `src/utils/systemPromptType.ts` | 品牌类型：`SystemPrompt`、`asSystemPrompt()` |
| `src/constants/system.ts` | 身份前缀：`getCLISyspromptPrefix()`、`getAttributionHeader()` |

### 上下文注入

| 文件 | 职责 |
|------|------|
| `src/context.ts` | `getUserContext()`（CLAUDE.md + 日期）、`getSystemContext()`（Git 状态）、`getGitStatus()` |
| `src/utils/claudemd.ts` | CLAUDE.md 文件发现、加载、@include 解析、优先级排序 |
| `src/memdir/memdir.ts` | MEMORY.md 自动记忆加载、`loadMemoryPrompt()` |
| `src/utils/api.ts` | `prependUserContext()`、`appendSystemContext()`、`splitSysPromptPrefix()`、`toolToAPISchema()` |
| `src/utils/queryContext.ts` | `fetchSystemPromptParts()` — 三件套（systemPrompt + userContext + systemContext）并行获取 |

### API 层

| 文件 | 职责 |
|------|------|
| `src/services/api/claude.ts` | `queryModel()`、`buildSystemPromptBlocks()`、`addCacheBreakpoints()`、身份前缀注入、prompt cache 控制 |
| `src/services/api/promptCacheBreakDetection.ts` | 缓存失效诊断 |
| `src/services/api/dumpPrompts.ts` | API 请求 dump（调试用） |
| `src/utils/toolSchemaCache.ts` | 工具 schema 会话级缓存 |

### 消息处理

| 文件 | 职责 |
|------|------|
| `src/utils/messages.ts` | 消息标准化、Plan 模式指令附件、`normalizeMessagesForAPI()` |
| `src/utils/processUserInput/processTextPrompt.ts` | 用户文本输入处理 |
| `src/utils/handlePromptSubmit.ts` | REPL 提交处理 |
| `src/query.ts` | 主查询循环，消息准备 + prompt 组装 + 模型调用 |

### Compact 系统

| 文件 | 职责 |
|------|------|
| `src/services/compact/prompt.ts` | Compact prompt 模板、`formatCompactSummary()` |
| `src/services/compact/compact.ts` | 完整 compact 流程 |
| `src/services/compact/autoCompact.ts` | 自动 compact 阈值和触发 |
| `src/services/compact/microCompact.ts` | 微压缩 |

### 工具 Prompt

| 文件 | 职责 |
|------|------|
| `src/tools/BashTool/prompt.ts` | Bash 工具描述 |
| `src/tools/FileReadTool/prompt.ts` | 文件读取工具描述 |
| `src/tools/FileEditTool/prompt.ts` | 文件编辑工具描述 |
| `src/tools/AgentTool/prompt.ts` | Agent 工具描述 |
| `src/tools/*/prompt.ts` | 各工具的描述文件 |

### 特殊 Prompt

| 文件 | 职责 |
|------|------|
| `src/constants/cyberRiskInstruction.ts` | 安全合规指令 `CYBER_RISK_INSTRUCTION` |
| `src/constants/outputStyles.ts` | 输出风格（Explanatory / Learning 模式）|
| `src/coordinator/coordinatorMode.ts` | Coordinator 专用 prompt |
| `src/tools/AgentTool/built-in/*.ts` | 内置 Agent 各自的系统 prompt |
| `src/utils/permissions/yoloClassifier.ts` | 自动模式分类器 prompt |

### A/B 实验

| 机制 | 说明 |
|------|------|
| GrowthBook | `getFeatureValue_CACHED_MAY_BE_STALE()` 控制各种 prompt 行为（验证合同、数值锚点等） |
| Statsig | `checkStatsigFeatureGate_CACHED_MAY_BE_STALE()` 控制 strict tools 等 |
| Feature flags | `feature('PROACTIVE')`、`feature('KAIROS')` 等编译时标记控制代码路径 |
| Ant-only | `process.env.USER_TYPE === 'ant'` 区分内部和外部用户的 prompt 差异 |
