---
title: Claude Code 工具系统
aliases: [工具系统, Tool System]
series: Claude Code 架构解析
category: 工具与Agent
order: 8
tags:
  - claude-code
  - tools
  - tool-execution
  - streaming
  - permissions
date: 2026-04-04
---

# Claude Code 工具系统 (Tool System) — 深度技术文档

## 目录

1. [架构概览](#1-架构概览)
2. [Tool 核心类型定义](#2-tool-核心类型定义)
   - 2.1 [三泛型参数签名](#21-三泛型参数签名)
   - 2.2 [必须实现的方法](#22-必须实现的方法)
   - 2.3 [可选扩展方法](#23-可选扩展方法)
   - 2.4 [渲染方法族](#24-渲染方法族)
3. [`buildTool()` 工厂函数](#3-buildtool-工厂函数)
4. [`ToolUseContext` — 工具运行时上下文](#4-toolusecontext--工具运行时上下文)
5. [`ToolResult` — 工具执行结果](#5-toolresult--工具执行结果)
6. [工具权限系统 (Permission System)](#6-工具权限系统-permission-system)
   - 6.1 [权限决策链](#61-权限决策链)
   - 6.2 [权限模式](#62-权限模式)
   - 6.3 [PermissionResult 结构](#63-permissionresult-结构)
   - 6.4 [权限决策原因](#64-权限决策原因-permissiondecisionreason)
   - 6.5 [精细权限匹配](#65-精细权限匹配)
   - 6.6 [权限上下文](#66-权限上下文-toolpermissioncontext)
7. [进度上报系统](#7-进度上报系统)
8. [工具注册表 (`tools.ts`)](#8-工具注册表-toolsts)
   - 8.1 [全量注册](#81-全量注册)
   - 8.2 [核心工具](#82-核心工具-始终可用)
   - 8.3 [条件加载工具](#83-条件加载工具-feature-flag--环境变量保护)
   - 8.4 [工具过滤流水线](#84-工具过滤流水线)
9. [ToolSearch 延迟加载](#9-toolsearch-延迟加载)
10. [MCP 工具 (Model Context Protocol)](#10-mcp-工具-model-context-protocol)
11. [Agent 工具与子 Agent 定义](#11-agent-工具与子-agent-定义)
    - 11.1 [Agent 定义字段](#111-agent-定义字段)
    - 11.2 [Markdown Agent 定义格式](#112-markdown-agent-定义格式)
    - 11.3 [覆盖优先级](#113-覆盖优先级)
12. [工具别名机制](#12-工具别名机制)
13. [设计模式总结](#13-设计模式总结)

---

## 1. 架构概览

Tool 系统采用**类型驱动的插件架构**，核心定义在 `src/Tool.ts`（~793 行）中，是 Claude Code 能力的基石。整个系统由三层组成：

```
┌─────────────────────────────────────────────────────────────┐
│                   Tool 接口 (src/Tool.ts)                    │
│  Tool<Input, Output, Progress> — 完整的 tool 契约            │
│  ToolDef<...>                 — 可省略默认方法的定义类型      │
│  buildTool(def)               — 注入默认实现的工厂函数        │
└─────────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────────┐
│              具体 Tool 实现 (src/tools/*/)                   │
│  BashTool / FileReadTool / AgentTool / MCPTool / ...        │
│  每个工具一个目录，包含实现、prompt、常量、UI 组件             │
└─────────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────────┐
│            Tool 注册表 (src/tools.ts)                        │
│  getAllBaseTools() — 全量工具列表（唯一注册点）               │
│  getTools(permCtx) — 按权限+环境过滤后的最终工具池            │
└─────────────────────────────────────────────────────────────┘
```

**关键源文件**：

| 文件 | 行数 | 职责 |
|------|------|------|
| `src/Tool.ts` | ~793 | 核心类型定义 + `buildTool()` 工厂 |
| `src/tools.ts` | ~390 | 工具注册表 + 过滤逻辑 |
| `src/tools/*/` | ~149 文件 | 40+ 工具实现 |
| `src/types/permissions.ts` | ~442 | 权限类型定义 |
| `src/hooks/useCanUseTool.tsx` | ~200 | 权限检查入口 Hook |
| `src/tools/AgentTool/loadAgentsDir.ts` | ~756 | Agent 定义加载 |

---

## 2. Tool 核心类型定义

### 2.1 三泛型参数签名

```typescript
export type Tool<
  Input extends AnyObject = AnyObject,   // Zod schema → JSON Schema 输入结构
  Output = unknown,                       // 工具执行结果类型
  P extends ToolProgressData = ToolProgressData,  // 进度事件类型
> = { ... }
```

- **Input**：基于 Zod v4 定义的输入 schema，自动转换为 JSON Schema 供 API 使用
- **Output**：`call()` 返回的 `ToolResult<Output>` 中的数据类型
- **P**：流式进度事件的具体类型（如 `BashProgress`、`AgentToolProgress`）

辅助类型：

```typescript
// 任何输出对象类型的 Zod schema
type AnyObject = z.ZodType<{ [key: string]: unknown }>

// 工具集合类型（readonly 数组，防止意外修改）
type Tools = readonly Tool[]
```

### 2.2 必须实现的方法

| 方法 | 签名 | 职责 |
|------|------|------|
| `call` | `(args, context, canUseTool, parentMessage, onProgress?) → Promise<ToolResult<Output>>` | **核心执行逻辑** |
| `description` | `(input, options) → Promise<string>` | 返回该次调用的人可读描述（权限弹窗显示） |
| `prompt` | `(options) → Promise<string>` | 生成注入 system prompt 的工具说明文档 |
| `inputSchema` | `Input`（readonly Zod schema） | 定义并校验输入参数 |
| `name` | `readonly string` | 工具唯一名称（不可重复） |
| `maxResultSizeChars` | `number` | 结果超过此字数后持久化到磁盘（`Infinity` 表示永不持久化） |
| `checkPermissions` | `(input, context) → Promise<PermissionResult>` | 工具级别的权限检查逻辑 |
| `mapToolResultToToolResultBlockParam` | `(content, toolUseID) → ToolResultBlockParam` | 序列化结果为 Anthropic API 的 `tool_result` 格式 |
| `renderToolUseMessage` | `(input, options) → React.ReactNode` | 渲染工具调用的终端 UI（参数可能为 `Partial`，因流式传输可能未完整） |
| `userFacingName` | `(input) → string` | 面向用户的展示名称 |
| `toAutoClassifierInput` | `(input) → unknown` | 自动模式安全分类器的输入摘要（安全相关工具必须覆盖） |
| `isConcurrencySafe` | `(input) → boolean` | 是否可与其它工具并行执行 |
| `isReadOnly` | `(input) → boolean` | 是否为只读操作 |
| `isEnabled` | `() → boolean` | 是否在当前环境可用 |

### 2.3 可选扩展方法

| 方法 | 签名 | 用途 |
|------|------|------|
| `aliases` | `string[]` | 工具改名后的向后兼容别名 |
| `searchHint` | `string` | ToolSearch 关键词匹配（3-10 词，如 NotebookEdit 填 `'jupyter'`） |
| `shouldDefer` | `readonly boolean` | 是否延迟加载（需通过 ToolSearch 发现后才暴露） |
| `alwaysLoad` | `readonly boolean` | 即使开启 defer 也始终出现在第一轮 prompt |
| `strict` | `readonly boolean` | 启用严格模式（API 更严格遵循工具指令和参数 schema） |
| `interruptBehavior` | `() → 'cancel' \| 'block'` | 用户提交新消息时：中止还是阻塞（默认 `'block'`） |
| `isDestructive` | `(input) → boolean` | 是否执行不可逆操作（delete/overwrite/send） |
| `isSearchOrReadCommand` | `(input) → { isSearch, isRead, isList? }` | UI 折叠判断（搜索/读取/列目录） |
| `isOpenWorld` | `(input) → boolean` | 是否有外部副作用 |
| `requiresUserInteraction` | `() → boolean` | 是否需要用户交互 |
| `isMcp` | `boolean` | 是否为 MCP 工具 |
| `isLsp` | `boolean` | 是否为 LSP 工具 |
| `mcpInfo` | `{ serverName, toolName }` | MCP 工具的服务器和原始工具名 |
| `validateInput` | `(input, context) → Promise<ValidationResult>` | 在权限检查**之前**校验输入合法性 |
| `preparePermissionMatcher` | `(input) → Promise<(pattern: string) → boolean>` | Hook `if` 条件的模式匹配器工厂 |
| `getPath` | `(input) → string` | 返回操作的文件路径 |
| `backfillObservableInput` | `(input: Record<string, unknown>) → void` | 为观察者（SDK 流/转录/hooks）补充派生字段（幂等） |
| `inputsEquivalent` | `(a, b) → boolean` | 判断两个输入是否等价（用于推测执行去重） |
| `isTransparentWrapper` | `() → boolean` | 透明包装器（如 REPL），委托所有渲染给进度处理器 |
| `getToolUseSummary` | `(input) → string \| null` | 紧凑视图中的摘要文本 |
| `getActivityDescription` | `(input) → string \| null` | Spinner 显示的活动描述（如 "Reading src/foo.ts"） |
| `extractSearchText` | `(out) → string` | 转录搜索索引的文本提取 |
| `isResultTruncated` | `(output) → boolean` | 非 verbose 模式下是否有截断（控制展开按钮） |

### 2.4 渲染方法族

每个工具在不同生命周期阶段提供对应的 React/Ink 渲染方法：

| 阶段 | 渲染方法 | 触发时机 | 必须? |
|------|---------|---------|-------|
| 工具调用开始 | `renderToolUseMessage` | 模型发出 tool_use block（参数可能尚未完整流入） | **是** |
| 工具执行中 | `renderToolUseProgressMessage` | 每收到 `onProgress` 事件 | 否 |
| 工具排队等待 | `renderToolUseQueuedMessage` | 工具在权限队列中等待用户确认 | 否 |
| 工具拒绝 | `renderToolUseRejectedMessage` | 用户拒绝权限请求 | 否（回退到 `FallbackToolUseRejectedMessage`） |
| 工具出错 | `renderToolUseErrorMessage` | 执行抛出异常 | 否（回退到 `FallbackToolUseErrorMessage`） |
| 工具完成 | `renderToolResultMessage` | 工具返回结果 | 否（省略则不渲染结果） |
| 附加标签 | `renderToolUseTag` | 工具调用后的元数据标签（如超时、模型名称） | 否 |
| 多工具分组 | `renderGroupedToolUse` | 多个同类工具并行时聚合显示 | 否（仅非 verbose 模式） |

**关键设计点**: `renderToolUseMessage` 的 `input` 为 `Partial<z.infer<Input>>`，因为模型参数是流式传入的，渲染时部分字段可能为 `undefined`，实现必须对缺失字段保持宽容。

**分组渲染**: `renderGroupedToolUse` 接收一组并行 tool_use 的完整信息（参数、状态、进度、结果），可返回聚合视图（如多个 Grep 结果合并展示），返回 `null` 则回退到逐个渲染。

---

## 3. `buildTool()` 工厂函数

`buildTool()` 实现了**安全默认值注入**模式，是所有工具的唯一构建入口。

### 3.1 运行时行为

```typescript
export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,
    userFacingName: () => def.name,
    ...def,
  } as BuiltTool<D>
}
```

### 3.2 默认值表（Fail-Closed 安全设计）

| 默认方法 | 默认值 | 设计理由 |
|---------|-------|---------|
| `isEnabled` | `() => true` | 工具默认启用 |
| `isConcurrencySafe` | `() => false` | **假设不安全**，避免并发竞态错误 |
| `isReadOnly` | `() => false` | **假设有写操作**，确保触发权限检查 |
| `isDestructive` | `() => false` | 非破坏性（执行不可逆操作的工具**必须覆盖**） |
| `checkPermissions` | `→ { behavior: 'allow' }` | 委托给通用权限系统处理 |
| `toAutoClassifierInput` | `→ ''` | 跳过分类器（安全相关工具**必须覆盖**） |
| `userFacingName` | `→ def.name` | 使用工具名作为显示名 |

### 3.3 类型层面精确性

```typescript
// ToolDef: 允许省略有默认值的方法
type ToolDef<Input, Output, P> =
  Omit<Tool<Input, Output, P>, DefaultableToolKeys> &
  Partial<Pick<Tool<Input, Output, P>, DefaultableToolKeys>>

// BuiltTool<D>: 在类型层面模拟 { ...TOOL_DEFAULTS, ...def }
// 如果 D 提供了某方法 → 用 D 的类型
// 如果 D 省略了某方法 → 用 TOOL_DEFAULTS 的类型
type BuiltTool<D> = Omit<D, DefaultableToolKeys> & {
  [K in DefaultableToolKeys]-?: K extends keyof D
    ? undefined extends D[K] ? ToolDefaults[K] : D[K]
    : ToolDefaults[K]
}
```

### 3.4 使用方式

```typescript
// 每个工具文件底部
export const FileReadTool = buildTool({
  name: FILE_READ_TOOL_NAME,
  maxResultSizeChars: Infinity,  // 读文件工具永不持久化（避免循环 Read→file→Read）
  isReadOnly: () => true,
  isConcurrencySafe: () => true,
  // ...具体实现
} satisfies ToolDef<InputSchema, Output>)
```

`satisfies ToolDef<...>` 确保编译时类型安全，同时 `buildTool` 的泛型推断精确保留了每个工具的具体类型信息。全代码库 60+ 个工具都通过此入口构建，确保默认值逻辑唯一。

---

## 4. `ToolUseContext` — 工具运行时上下文

`ToolUseContext` 是传递给每次 `call()` 的"万能上下文"，包含工具执行所需的所有基础设施。按职责分为 7 组：

```typescript
export type ToolUseContext = {
  // ═══════════════ 配置选项 ═══════════════
  options: {
    commands: Command[]           // 已注册的斜杠命令
    tools: Tools                   // 完整工具池
    mainLoopModel: string         // 主循环模型名称
    mcpClients: MCPServerConnection[]  // MCP 服务器连接
    mcpResources: Record<string, ServerResource[]>
    isNonInteractiveSession: boolean
    agentDefinitions: AgentDefinitionsResult
    maxBudgetUsd?: number         // USD 预算上限
    customSystemPrompt?: string   // 自定义系统提示词
    appendSystemPrompt?: string   // 追加系统提示词
    refreshTools?: () => Tools    // 动态刷新工具（MCP 热连接）
    thinkingConfig: ThinkingConfig
    verbose: boolean
    debug: boolean
  }

  // ═══════════════ 运行控制 ═══════════════
  abortController: AbortController     // 取消信号
  messages: Message[]                   // 完整对话历史
  readFileState: FileStateCache        // 文件读取缓存（跨 Turn 保留）
  fileReadingLimits?: { maxTokens?, maxSizeBytes? }
  globLimits?: { maxResults? }

  // ═══════════════ 状态管理 ═══════════════
  getAppState(): AppState              // 读取全局状态
  setAppState(f: (prev: AppState) => AppState): void  // 更新全局状态
  setAppStateForTasks?: (...)          // 子 Agent 也能触达根 store
  updateFileHistoryState: (...)        // 文件变更历史追踪
  updateAttributionState: (...)        // commit attribution 追踪
  contentReplacementState?: ContentReplacementState  // 结果预算管理

  // ═══════════════ UI 交互 ═══════════════
  setToolJSX?: SetToolJSXFn            // REPL 模式：注入自定义 JSX
  addNotification?: (notif) => void    // 系统通知
  sendOSNotification?: (opts) => void  // OS 级通知（iTerm2/Kitty/Ghostty）
  appendSystemMessage?: (msg) => void  // 追加 UI 系统消息（在 API 边界被剥离）
  setStreamMode?: (mode: SpinnerMode) => void
  openMessageSelector?: () => void
  setConversationId?: (id: UUID) => void

  // ═══════════════ 执行追踪 ═══════════════
  setInProgressToolUseIDs: (f: (prev: Set<string>) => Set<string>) => void
  setHasInterruptibleToolInProgress?: (v: boolean) => void  // 仅 REPL 模式
  setResponseLength: (f: (prev: number) => number) => void
  pushApiMetricsEntry?: (ttftMs: number) => void  // OTPS 追踪

  // ═══════════════ Agent 信息 ═══════════════
  agentId?: AgentId           // 仅子 Agent 有
  agentType?: string          // 子 Agent 类型名
  queryTracking?: QueryChainTracking  // { chainId, depth }

  // ═══════════════ 权限相关 ═══════════════
  toolDecisions?: Map<string, { source, decision, timestamp }>
  localDenialTracking?: DenialTrackingState  // 异步子 Agent 的本地拒绝追踪
  requireCanUseTool?: boolean  // 推测执行的路径重写

  // ═══════════════ 技能/记忆/Hook ═══════════════
  nestedMemoryAttachmentTriggers?: Set<string>
  loadedNestedMemoryPaths?: Set<string>  // 防重复注入 CLAUDE.md
  dynamicSkillDirTriggers?: Set<string>
  discoveredSkillNames?: Set<string>     // 遥测用
  requestPrompt?: (sourceName, toolInputSummary?) => (request) => Promise<PromptResponse>
  handleElicitation?: (serverName, params, signal) => Promise<ElicitResult>
  renderedSystemPrompt?: SystemPrompt    // Fork 子 Agent 缓存共享
}
```

**设计要点**：

- **`setAppStateForTasks`**：解决了子 Agent 的 `setAppState` 被 `createSubagentContext` 设为 no-op 的问题，让深层嵌套的 Agent 也能注册/清理 session 级基础设施（如后台任务）
- **`contentReplacementState`**：在主线程创建后永不重置（过期键无害），子 Agent 通过 clone 共享缓存决策
- **`renderedSystemPrompt`**：冻结了 fork 时刻的系统提示词，避免 GrowthBook 冷→热状态变化导致 prompt cache bust
- **`loadedNestedMemoryPaths`**：`readFileState` 是 LRU 缓存会淘汰条目，单独用 `Set<string>` 去重防止重复注入同一个 CLAUDE.md

---

## 5. `ToolResult` — 工具执行结果

```typescript
export type ToolResult<T> = {
  data: T                    // 结果主体
  newMessages?: (            // 注入对话历史的额外消息
    | UserMessage | AssistantMessage | AttachmentMessage | SystemMessage
  )[]
  contextModifier?: (context: ToolUseContext) => ToolUseContext  // 修改后续上下文
  mcpMeta?: {               // MCP 协议元数据透传给 SDK 消费方
    _meta?: Record<string, unknown>
    structuredContent?: Record<string, unknown>
  }
}
```

**字段详解**：

- **`data`**：工具执行结果主体。经 `mapToolResultToToolResultBlockParam()` 序列化后注入 API 消息流
- **`newMessages`**：工具可以注入额外消息到对话历史（如 FileReadTool 注入文件内容作为 `AttachmentMessage`）
- **`contextModifier`**：修改后续工具调用的上下文。**仅对 `isConcurrencySafe() === false` 的工具有效**（并发安全工具的 contextModifier 被忽略）
- **`mcpMeta`**：MCP `structuredContent` / `_meta` 透传字段

**结果持久化**：当 `data` 序列化后超过 `maxResultSizeChars` 时，内容被写入临时文件，模型收到文件路径 + 预览摘要（通过 `ContentReplacementState` 追踪）。`FileReadTool` 设置 `maxResultSizeChars: Infinity` 以豁免此机制，避免循环 `Read→file→Read`。

---

## 6. 工具权限系统 (Permission System)

### 6.1 权限决策链

权限系统是多层决策链，每层返回 `PermissionResult`：

```
工具调用
  │
  ▼
validateInput()  ── 输入合法性校验（如文件路径存在性）
  │ 通过
  ▼
checkPermissions()  ── 工具自身的权限判断
  │
  ├── behavior='allow'  → 直接放行
  ├── behavior='deny'   → 直接拒绝
  ├── behavior='ask'    → 需要用户确认
  └── behavior='passthrough'  → 委托通用系统（MCP 工具常用）
        │
        ▼
  hasPermissionsToUseTool()  ── 通用权限系统
        │
        ├── 1. 权限模式检查 (bypassPermissions → 全部放行)
        ├── 2. alwaysDenyRules 匹配?  → deny
        ├── 3. alwaysAllowRules 匹配? → allow
        ├── 4. alwaysAskRules 匹配?   → ask
        ├── 5. PreToolUse Hook 评估
        ├── 6. YoloClassifier (auto mode 专属)
        └── 7. interactive prompt (REPL 模式 → 弹窗询问用户)
```

### 6.2 权限模式

```typescript
// 外部可见模式（用户可设置）
'acceptEdits'      // 自动接受编辑，其他操作仍需确认
'bypassPermissions' // 跳过所有权限检查
'default'          // 标准模式，按规则判断
'dontAsk'          // 不询问，直接拒绝不确定项
'plan'             // 规划模式，只允许只读操作

// 内部扩展模式
'auto'             // 自动模式 — 使用分类器替代人工审批（Feature Flag 保护）
'bubble'           // 冒泡到父 Agent 处理
```

### 6.3 PermissionResult 结构

```typescript
type PermissionResult<Input> =
  | { behavior: 'allow';  updatedInput?: Input; decisionReason?: ... }
  | { behavior: 'ask';    message: string; suggestions?: PermissionUpdate[];
      pendingClassifierCheck?: PendingClassifierCheck; ... }
  | { behavior: 'deny';   message: string; decisionReason: PermissionDecisionReason }
  | { behavior: 'passthrough'; message: string; ... }  // MCP 工具特有
```

- **`allow`**：允许执行，可附带修改后的 `updatedInput`（如路径规范化）
- **`ask`**：需要用户确认，附带建议的权限更新（如"总是允许 git 命令"）和可选的异步分类器检查
- **`deny`**：拒绝执行，必须附带原因
- **`passthrough`**：MCP 工具独有 — 由于 MCP 工具无法在定义时知道自己的权限语义，它返回 `passthrough` 让通用权限系统全权处理

### 6.4 权限决策原因 (PermissionDecisionReason)

每个权限决定都携带原因，用于审计和 UI 展示：

```typescript
type PermissionDecisionReason =
  | { type: 'rule';           rule: PermissionRule }                // 命中显式规则
  | { type: 'mode';           mode: PermissionMode }                // 当前权限模式
  | { type: 'hook';           hookName, hookSource?, reason? }      // PreToolUse hook
  | { type: 'classifier';     classifier, reason }                  // 自动分类器
  | { type: 'sandboxOverride'; reason: 'excludedCommand' | 'dangerouslyDisableSandbox' }
  | { type: 'workingDir';     reason }                              // 工作目录限制
  | { type: 'safetyCheck';    reason, classifierApprovable }        // 安全检查
  | { type: 'subcommandResults'; reasons: Map<string, PermissionResult> } // Bash 子命令聚合
  | { type: 'asyncAgent';     reason }                              // 异步 Agent
  | { type: 'permissionPromptTool'; permissionPromptToolName, toolResult } // 自定义权限工具
  | { type: 'other';          reason }                              // 兜底
```

### 6.5 精细权限匹配

工具可通过 `preparePermissionMatcher()` 提供模式匹配器，用于 Hook `if` 条件评估：

```typescript
// 在 BashTool 中：支持 "Bash(git *)" 风格的规则
preparePermissionMatcher(input) {
  return async (pattern: string) => {
    // input.command = "git commit -m 'fix'" 匹配规则 "git *" → true
    return matchWildcardPattern(input.command, pattern)
  }
}
```

这允许在 `.claude/settings.json` 或 hooks 中写出精细的权限规则：

```json
{
  "permissions": {
    "allow": ["Bash(git *)", "Bash(npm test)", "Read"],
    "deny": ["Bash(rm -rf *)"]
  }
}
```

### 6.6 权限上下文 (ToolPermissionContext)

```typescript
type ToolPermissionContext = DeepImmutable<{
  mode: PermissionMode
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  alwaysAllowRules: ToolPermissionRulesBySource   // 按来源分组的允许规则
  alwaysDenyRules: ToolPermissionRulesBySource    // 按来源分组的拒绝规则
  alwaysAskRules: ToolPermissionRulesBySource     // 按来源分组的询问规则
  isBypassPermissionsModeAvailable: boolean
  shouldAvoidPermissionPrompts?: boolean          // 后台 Agent 无 UI 时自动拒绝
  awaitAutomatedChecksBeforeDialog?: boolean      // Coordinator worker 等待自动检查
  prePlanMode?: PermissionMode                    // 进入 plan mode 前的模式（用于恢复）
}>
```

整个上下文使用 `DeepImmutable<>` 包裹，防止运行时意外修改。

规则来源优先级（后者覆盖前者）：
`userSettings` → `projectSettings` → `localSettings` → `flagSettings` → `policySettings` → `cliArg` → `command` → `session`

---

## 7. 进度上报系统

工具执行时可通过 `onProgress` 回调流式上报进度：

```typescript
type ToolProgress<P extends ToolProgressData> = {
  toolUseID: string
  data: P
}

type ToolCallProgress<P extends ToolProgressData> = (
  progress: ToolProgress<P>,
) => void
```

每种工具有专属的 Progress 结构：

| 类型 | 工具 | 典型数据 |
|------|------|---------|
| `BashProgress` | BashTool | stdout/stderr 流、退出码 |
| `AgentToolProgress` | AgentTool | 子 Agent 消息流、token 用量 |
| `MCPProgress` | MCPTool | MCP 调用状态 |
| `REPLToolProgress` | REPLTool | REPL 沙箱内工具进度（转发内部工具的进度类型） |
| `SkillToolProgress` | SkillTool | Skill 调用进度 |
| `WebSearchProgress` | WebSearchTool | 搜索结果流 |
| `TaskOutputProgress` | TaskOutputTool | 后台 Task 输出 |

进度事件与 Hook 事件的区分：

```typescript
type Progress = ToolProgressData | HookProgress

// 过滤出纯工具进度（排除 hook 进度）
function filterToolProgressMessages(msgs): ProgressMessage<ToolProgressData>[]
```

---

## 8. 工具注册表 (`tools.ts`)

### 8.1 全量注册

`getAllBaseTools()` 是所有工具的唯一注册点（~250 行）：

```typescript
export function getAllBaseTools(): Tools {
  return [
    AgentTool, TaskOutputTool, BashTool,
    // Embedded search 模式下禁用（Ant 二进制内嵌了 bfs/ugrep）
    ...(hasEmbeddedSearchTools() ? [] : [GlobTool, GrepTool]),
    ExitPlanModeV2Tool, FileReadTool, FileEditTool, FileWriteTool,
    NotebookEditTool, WebFetchTool, TodoWriteTool, WebSearchTool,
    TaskStopTool, AskUserQuestionTool, SkillTool, EnterPlanModeTool,
    // Feature-flagged tools...
    ...(isToolSearchEnabledOptimistic() ? [ToolSearchTool] : []),
  ]
}
```

> 注：此列表必须与 Statsig 上的 `claude_code_global_system_caching` 配置保持同步，以确保跨用户的 system prompt 缓存命中。

### 8.2 核心工具 (始终可用)

| 工具 | 用途 |
|------|------|
| `BashTool` | 执行 Shell 命令 |
| `FileReadTool` | 读取文件（支持图片/PDF/Notebook/范围读取） |
| `FileEditTool` | 编辑文件（精确字符串替换） |
| `FileWriteTool` | 写入/创建文件 |
| `GlobTool` | 文件名模式搜索 |
| `GrepTool` | 基于 ripgrep 的内容搜索 |
| `WebFetchTool` | 获取网页内容（转 markdown） |
| `WebSearchTool` | 网络搜索 |
| `AgentTool` | 启动子 Agent |
| `TaskOutputTool` | 获取后台 Task 输出 |
| `TaskStopTool` | 停止后台 Task |
| `TodoWriteTool` | 任务管理 |
| `SkillTool` | 技能调用 |
| `NotebookEditTool` | Jupyter Notebook 编辑 |
| `AskUserQuestionTool` | 向用户提问 |
| `EnterPlanModeTool` / `ExitPlanModeV2Tool` | 规划模式切换 |
| `BriefTool` | 简报工具 |
| `SendMessageTool` | 向协作 Agent 发消息 |
| `ListMcpResourcesTool` / `ReadMcpResourceTool` | MCP 资源管理 |

### 8.3 条件加载工具 (Feature Flag / 环境变量保护)

| 工具 | 条件 | 用途 |
|------|------|------|
| `SleepTool` | `PROACTIVE` / `KAIROS` | 定时等待 |
| `CronCreate/Delete/ListTool` | `AGENT_TRIGGERS` | Cron 定时任务 |
| `RemoteTriggerTool` | `AGENT_TRIGGERS_REMOTE` | 远程触发 |
| `MonitorTool` | `MONITOR_TOOL` | 日志监控 |
| `WebBrowserTool` | `WEB_BROWSER_TOOL` | 浏览器自动化 |
| `SnipTool` | `HISTORY_SNIP` | 历史裁剪 |
| `WorkflowTool` | `WORKFLOW_SCRIPTS` | 工作流 |
| `ToolSearchTool` | 动态检测 | 工具搜索/延迟加载 |
| `EnterWorktreeTool` / `ExitWorktreeTool` | Worktree 模式 | Git worktree 隔离 |
| `TeamCreateTool` / `TeamDeleteTool` | Agent Swarms | 多 Agent 团队 |
| `SubscribePRTool` | `KAIROS_GITHUB_WEBHOOKS` | PR 订阅 |
| `PushNotificationTool` | `KAIROS` / `KAIROS_PUSH_NOTIFICATION` | 推送通知 |
| `REPLTool` | `USER_TYPE=ant` | REPL 虚拟机沙箱 |
| `PowerShellTool` | Windows | PowerShell |
| `CtxInspectTool` | `CONTEXT_COLLAPSE` | 上下文检查 |
| `ListPeersTool` | `UDS_INBOX` | 对等 Agent 发现 |
| `LSPTool` | `ENABLE_LSP_TOOL` | 语言服务器协议 |
| `ConfigTool` / `TungstenTool` | `USER_TYPE=ant` | 内部配置/Tungsten |
| `VerifyPlanExecutionTool` | `CLAUDE_CODE_VERIFY_PLAN=true` | 计划执行验证 |
| `SuggestBackgroundPRTool` | `USER_TYPE=ant` | 建议后台 PR |
| `SendUserFileTool` | `KAIROS` | 发送用户文件 |
| `OverflowTestTool` | `OVERFLOW_TEST_TOOL` | 溢出测试 |
| `TerminalCaptureTool` | `TERMINAL_PANEL` | 终端面板捕获 |

条件加载利用 Bun bundler 的 `feature()` 宏实现**编译时死代码消除**，确保外部构建中不包含内部功能代码。

### 8.4 工具过滤流水线

```typescript
getTools(permissionContext) {
  // 1. 简单模式：最小工具集
  if (CLAUDE_CODE_SIMPLE) → [BashTool, FileReadTool, FileEditTool]
    // Coordinator 模式额外加 AgentTool + TaskStopTool + SendMessageTool

  // 2. REPL 模式：用 REPLTool 替换原始工具
  if (isReplModeEnabled()) → [REPLTool, ...non-REPL-only-tools]
    // REPL 内部在 VM 沙箱中暴露原始工具

  // 3. 常规模式：全量工具
  getAllBaseTools()

  // 4. 基于 deny 规则过滤（对模型完全不可见）
  filterToolsByDenyRules(tools, permissionContext)
  // MCP server 前缀规则（如 "mcp__server"）可批量移除整个 server 的工具

  // 5. 合并 MCP 工具，按名称去重
  uniqBy([...baseTools, ...mcpTools], 'name')
}
```

`filterToolsByDenyRules` 的核心逻辑：如果某个工具匹配到了 deny 规则中**没有 `ruleContent`** 的条目（即全量拒绝该工具），则从候选池中完全移除，模型甚至看不到该工具的存在。

---

## 9. ToolSearch 延迟加载

为减少 system prompt 的 token 用量（40+ 工具的 prompt 可能消耗大量上下文），支持**延迟加载（defer）**机制：

### 9.1 工作流程

```
初始 prompt 仅包含：
  ┌─ alwaysLoad=true 的工具（完整 schema + prompt）
  ├─ shouldDefer=true 的工具（仅 defer_loading: true 标记，无 schema）
  └─ ToolSearchTool 自身（完整 schema）

模型需要使用被延迟的工具时：
  1. 调用 ToolSearchTool(query="关键词")
  2. ToolSearchTool 用关键词匹配 searchHint 和工具名称
  3. 匹配到的工具 schema 被动态注入后续 prompt
  4. 模型可以正常调用被激活的工具
```

### 9.2 searchHint 设计

工具通过 `searchHint` 提供 3-10 个辅助关键词（不与工具名重复）：

```typescript
// NotebookEditTool
searchHint: 'jupyter'

// WebBrowserTool
searchHint: 'browser automation puppeteer'
```

### 9.3 启用条件

```typescript
// 乐观检查 — 构建工具池时使用
isToolSearchEnabledOptimistic()

// 实际决策 — 每次请求时在 claude.ts 中决定
// 受 Feature Flag 和工具数量影响
```

---

## 10. MCP 工具 (Model Context Protocol)

MCP 工具通过 `MCPTool` 模板动态创建。`MCPTool.ts` 定义了一个**壳工具**，其核心方法在 `services/mcp/mcpClient.ts` 中被实际 MCP 服务器信息覆盖。

### 10.1 模板定义

```typescript
export const MCPTool = buildTool({
  isMcp: true,
  name: 'mcp',                           // 被覆盖为 mcp__<server>__<tool>
  maxResultSizeChars: 100_000,
  async call() { return { data: '' } },  // 被覆盖为实际 MCP 调用
  async checkPermissions() {
    return { behavior: 'passthrough', message: 'MCPTool requires permission.' }
  },
  userFacingName: () => 'mcp',           // 被覆盖
  // ...
})
```

### 10.2 运行时覆盖

```typescript
// mcpClient.ts 中动态创建每个 MCP 工具
const tool = { ...MCPTool }
tool.name = `mcp__${serverName}__${toolName}`
tool.mcpInfo = { serverName, toolName }
tool.call = async (args, context) => { /* 实际 MCP RPC 调用 */ }
tool.description = async () => mcpToolDescription
tool.userFacingName = () => `mcp: ${serverName}/${toolName}`
```

### 10.3 特殊设计

- **`inputSchema`**：使用 `z.object({}).passthrough()` 允许任意输入
- **`inputJSONSchema`**：可直接提供 JSON Schema 格式（绕过 Zod→JSON Schema 转换）
- **`checkPermissions`**：返回 `passthrough`，完全委托给通用权限系统
- **`_meta['anthropic/alwaysLoad']`**：MCP 服务器可标记工具为 `alwaysLoad`，使其绕过延迟加载
- **命名格式**：`mcp__<serverName>__<toolName>`（或在 `CLAUDE_AGENT_SDK_MCP_NO_PREFIX` 模式下无前缀）

---

## 11. Agent 工具与子 Agent 定义

`AgentTool` 负责派生子 Agent，子 Agent 定义（`AgentDefinition`）从三个来源加载：

```typescript
type AgentDefinition =
  | BuiltInAgentDefinition   // 硬编码在 builtInAgents.ts 中
  | CustomAgentDefinition    // 来自 agents/ 目录的 .md 或 JSON
  | PluginAgentDefinition    // 来自插件系统
```

类型判断：

```typescript
isBuiltInAgent(agent)  // agent.source === 'built-in'
isCustomAgent(agent)   // agent.source !== 'built-in' && !== 'plugin'
isPluginAgent(agent)   // agent.source === 'plugin'
```

### 11.1 Agent 定义字段

```typescript
type BaseAgentDefinition = {
  agentType: string            // 唯一标识（如 'explore', 'plan'）
  whenToUse: string            // 描述何时使用该 Agent（模型选择依据）
  tools?: string[]             // 白名单工具列表
  disallowedTools?: string[]   // 黑名单工具列表
  skills?: string[]            // 预加载的 Skill 名称
  mcpServers?: AgentMcpServerSpec[]  // Agent 专属 MCP 服务器
  hooks?: HooksSettings        // Agent 级别 hooks
  color?: AgentColorName       // UI 颜色
  model?: string               // 指定模型（或 'inherit' 继承父级）
  effort?: EffortValue         // 推理努力程度
  permissionMode?: PermissionMode
  maxTurns?: number            // 最大 agentic turn 数
  background?: boolean         // 始终作为后台任务运行
  initialPrompt?: string       // 首轮 user turn 前置内容（支持斜杠命令）
  memory?: 'user' | 'project' | 'local'  // 持久化记忆范围
  isolation?: 'worktree' | 'remote'      // 隔离模式（remote 仅 Ant 内部）
  omitClaudeMd?: boolean       // 省略 CLAUDE.md 层级（节省 token）
  requiredMcpServers?: string[]  // 必须存在的 MCP 服务器（模式匹配）
}
```

MCP 服务器规格支持两种形式：

```typescript
type AgentMcpServerSpec =
  | string                              // 引用已有服务器名（如 "slack"）
  | { [name: string]: McpServerConfig } // 内联定义
```

### 11.2 Markdown Agent 定义格式

```yaml
---
name: explore
description: Fast read-only codebase exploration agent
tools: [Read, Glob, Grep, Bash]
model: inherit
effort: low
color: cyan
maxTurns: 10
omitClaudeMd: true
---
You are a specialized exploration agent focused on quickly
finding relevant code and answering questions about the codebase...
```

JSON 格式定义同样支持：

```json
{
  "explore": {
    "description": "Fast read-only codebase exploration agent",
    "tools": ["Read", "Glob", "Grep", "Bash"],
    "model": "inherit",
    "effort": "low",
    "prompt": "You are a specialized exploration agent..."
  }
}
```

### 11.3 覆盖优先级

同名 Agent 的覆盖链（后出现的完全覆盖前者，通过 `Map.set()` 语义）：

```
built-in → plugin → userSettings → projectSettings → flagSettings → policySettings
```

加载流程：

```typescript
getAgentDefinitionsWithOverrides(cwd) {
  // 1. 加载 Markdown agent 文件
  const markdownFiles = await loadMarkdownFilesForSubdir('agents', cwd)
  const customAgents = markdownFiles.map(parseAgentFromMarkdown).filter(Boolean)

  // 2. 并行加载插件 agents + 初始化记忆快照
  const [pluginAgents] = await Promise.all([
    loadPluginAgents(),
    initializeAgentMemorySnapshots(customAgents),
  ])

  // 3. 获取内建 agents
  const builtInAgents = getBuiltInAgents()

  // 4. 按优先级合并，同名后者覆盖前者
  const allAgents = [...builtInAgents, ...pluginAgents, ...customAgents]
  const activeAgents = getActiveAgentsFromList(allAgents)  // Map 去重

  // 5. 为活跃 agents 初始化 UI 颜色
  for (const agent of activeAgents) {
    if (agent.color) setAgentColor(agent.agentType, agent.color)
  }
}
```

### 11.4 MCP 服务器要求过滤

```typescript
// 只有满足 MCP 服务器要求的 agent 才会出现在可用列表中
hasRequiredMcpServers(agent, availableServers)  // 模式匹配，大小写不敏感
filterAgentsByMcpRequirements(agents, availableServers)
```

---

## 12. 工具别名机制

工具支持别名，用于工具改名后的向后兼容：

```typescript
export function toolMatchesName(
  tool: { name: string; aliases?: string[] },
  name: string,
): boolean {
  return tool.name === name || (tool.aliases?.includes(name) ?? false)
}

export function findToolByName(tools: Tools, name: string): Tool | undefined {
  return tools.find(t => toolMatchesName(t, name))
}
```

查找时同时匹配主名和别名，确保旧版 API 调用在工具改名后仍能正常工作。

---

## 13. 设计模式总结

| 设计模式 | 体现 | 源文件 |
|---------|------|--------|
| **工厂方法 (Factory)** | `buildTool()` 注入安全默认值，统一构建入口 | `Tool.ts` |
| **策略模式 (Strategy)** | 每个工具实现自己的 `call()`、`checkPermissions()`、`renderToolResultMessage()` | `tools/*/` |
| **责任链 (Chain of Responsibility)** | 权限检查：工具级 → 通用规则 → Hook → 分类器 → 用户交互 | `permissions.ts` |
| **观察者 (Observer)** | `onProgress` 回调流式上报执行状态 | `Tool.ts` |
| **模板方法 (Template Method)** | `MCPTool` 作为壳模板，属性在运行时被 MCP 客户端覆盖 | `MCPTool.ts` / `mcpClient.ts` |
| **延迟加载 (Lazy Loading)** | `shouldDefer` + `ToolSearchTool` 按需暴露工具定义 | `ToolSearchTool.ts` |
| **Fail-Closed 安全** | 所有默认值均为最严格的安全假设（不并发安全、非只读） | `Tool.ts` TOOL_DEFAULTS |
| **类型层面展开 (Type-Level Spread)** | `BuiltTool<D>` 在类型层面精确模拟 `{...defaults, ...def}` | `Tool.ts` |
| **不可变上下文** | `ToolPermissionContext` 使用 `DeepImmutable<>` 防止意外修改 | `Tool.ts` |
| **条件编译 (Dead Code Elimination)** | `feature()` 宏 + `process.env.USER_TYPE` 编译时消除 | `tools.ts` |
| **适配器 (Adapter)** | MCP JSON Schema → Zod schema 适配，`inputJSONSchema` 旁路 | `MCPTool.ts` |
