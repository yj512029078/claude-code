---
title: Claude Code 对话管理系统
aliases: [对话管理, Conversation Management]
series: Claude Code 架构解析
category: 核心运行时
order: 5
tags:
  - claude-code
  - conversation
  - state-machine
  - compression
  - session
date: 2026-04-04
---

# Claude Code 对话管理系统 — 技术文档

## 目录

1. [架构概览](#1-架构概览)
2. [消息类型系统](#2-消息类型系统)
   - 2.1 [核心消息类型](#21-核心消息类型)
   - 2.2 [序列化消息 (SerializedMessage)](#22-序列化消息-serializedmessage)
   - 2.3 [转录消息 (TranscriptMessage)](#23-转录消息-transcriptmessage)
   - 2.4 [Entry 联合类型：JSONL 行的完整分类](#24-entry-联合类型jsonl-行的完整分类)
   - 2.5 [消息过滤谓词](#25-消息过滤谓词)
3. [对话状态的内存管理](#3-对话状态的内存管理)
   - 3.1 [REPL 路径：React 状态](#31-repl-路径react-状态)
   - 3.2 [SDK/Headless 路径：QueryEngine](#32-sdkheadless-路径queryengine)
   - 3.3 [两条路径的统一点](#33-两条路径的统一点)
4. [核心查询循环 (queryLoop)](#4-核心查询循环-queryloop)
   - 4.1 [循环状态 (State)](#41-循环状态-state)
   - 4.2 [消息预处理管道](#42-消息预处理管道)
   - 4.3 [API 调用与流式响应](#43-api-调用与流式响应)
   - 4.4 [工具执行与结果注入](#44-工具执行与结果注入)
   - 4.5 [循环续行与终止条件](#45-循环续行与终止条件)
   - 4.6 [完整消息流转路径](#46-完整消息流转路径)
5. [本地持久化：JSONL 存储](#5-本地持久化jsonl-存储)
   - 5.1 [文件结构与路径](#51-文件结构与路径)
   - 5.2 [Project 类：写入引擎](#52-project-类写入引擎)
   - 5.3 [写入队列与批量刷新](#53-写入队列与批量刷新)
   - 5.4 [延迟物化：避免空文件](#54-延迟物化避免空文件)
   - 5.5 [appendEntry 调度逻辑](#55-appendentry-调度逻辑)
   - 5.6 [insertMessageChain：消息链写入](#56-insertmessagechain消息链写入)
   - 5.7 [增量写入 Hook (useLogMessages)](#57-增量写入-hook-uselogmessages)
   - 5.8 [recordTranscript：去重与链维护](#58-recordtranscript去重与链维护)
   - 5.9 [禁用持久化的条件](#59-禁用持久化的条件)
6. [云端存储：远程持久化](#6-云端存储远程持久化)
   - 6.1 [Session Ingress v1](#61-session-ingress-v1)
   - 6.2 [CCR v2 内部事件通道](#62-ccr-v2-内部事件通道)
   - 6.3 [远端恢复 (Hydration)](#63-远端恢复-hydration)
   - 6.4 [本地与云端的关系](#64-本地与云端的关系)
7. [对话恢复 (Resume)](#7-对话恢复-resume)
   - 7.1 [日志加载与链重建](#71-日志加载与链重建)
   - 7.2 [消息反序列化](#72-消息反序列化)
   - 7.3 [中断检测与自动续行](#73-中断检测与自动续行)
   - 7.4 [附属状态恢复](#74-附属状态恢复)
8. [上下文窗口管理](#8-上下文窗口管理)
   - 8.1 [模型上下文窗口与 token 上限](#81-模型上下文窗口与-token-上限)
   - 8.2 [自动压缩 (Auto Compact)](#82-自动压缩-auto-compact)
   - 8.3 [工具结果预算 (Tool Result Budget)](#83-工具结果预算-tool-result-budget)
   - 8.4 [阻塞限制与预防性拒绝](#84-阻塞限制与预防性拒绝)
   - 8.5 [Token 预算追踪](#85-token-预算追踪)
9. [子 Agent 对话管理](#9-子-agent-对话管理)
10. [其他数据通道](#10-其他数据通道)
11. [关键源文件索引](#11-关键源文件索引)

---

## 1. 架构概览

Claude Code 的对话管理是一个**分层的、双路径的、具备远程持久化能力的系统**。它解决的核心问题是：在有限的 LLM 上下文窗口中驱动一个可能无限长的多轮 agentic 对话，同时保证对话可中断、可恢复、可跨设备同步。

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          用户输入入口                                    │
│  ┌────────────────────┐          ┌─────────────────────────┐           │
│  │ REPL (React state) │          │ QueryEngine (SDK/Headless)│          │
│  │ useState<Message[]>│          │ mutableMessages: Message[]│          │
│  └────────┬───────────┘          └───────────┬─────────────┘           │
│           │                                  │                         │
│           └──────────────┬───────────────────┘                         │
│                          ▼                                              │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │              processUserInput()                                  │   │
│  │  斜杠命令解析 → 附件处理 → 权限上下文 → UserMessage 构建         │   │
│  └────────────────────────┬────────────────────────────────────────┘   │
│                           ▼                                             │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │              queryLoop() — while(true) 主循环                    │   │
│  │                                                                   │  │
│  │  ┌───────────────────────────────────────────────────────┐       │  │
│  │  │ 消息预处理管道                                         │       │  │
│  │  │  Compact Boundary → Tool Budget → Snip → Microcompact │       │  │
│  │  │  → Context Collapse → Auto Compact                    │       │  │
│  │  └───────────────────────┬───────────────────────────────┘       │  │
│  │                          ▼                                        │  │
│  │  ┌───────────────────────────────────────────────────────┐       │  │
│  │  │ API 调用 (callModel / queryModelWithStreaming)         │       │  │
│  │  │  messages + systemPrompt + userContext → stream         │       │  │
│  │  └───────────────────────┬───────────────────────────────┘       │  │
│  │                          ▼                                        │  │
│  │  ┌───────────────────────────────────────────────────────┐       │  │
│  │  │ 流式响应 → AssistantMessage[] + tool_use blocks        │       │  │
│  │  └───────────────────────┬───────────────────────────────┘       │  │
│  │                          ▼                                        │  │
│  │  ┌───────────────────────────────────────────────────────┐       │  │
│  │  │ 工具执行 → toolResults (UserMessage[])                 │       │  │
│  │  └───────────────────────┬───────────────────────────────┘       │  │
│  │                          ▼                                        │  │
│  │  state.messages = [...messagesForQuery, ...assistant, ...tools]  │  │
│  │  turnCount++  →  continue while(true)                            │  │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                          持久化层                                        │
│  ┌──────────────────────┐     ┌──────────────────────────────────┐     │
│  │ 本地 JSONL            │     │ 远端 (Session Ingress / CCR v2)   │     │
│  │ ~/.claude/projects/   │◄───►│ 可选双写 + Hydration 恢复         │     │
│  │   <project>/          │     │                                    │     │
│  │   <sessionId>.jsonl   │     │                                    │     │
│  └──────────────────────┘     └──────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────────────┘
```

核心设计原则：

- **双入口统一引擎**：REPL 和 SDK 共享同一个 `query()` / `queryLoop()` 推理循环
- **Append-only 日志**：JSONL 只追加不修改，通过 `parentUuid` 链构建对话树
- **渐进式压缩**：6 层压缩策略在上下文逼近窗口极限时逐级介入
- **可选远程镜像**：本地 JSONL 是真相之源，云端是实时副本
- **确定性恢复**：从 JSONL 链重建完整对话状态，包括工具结果、权限、技能等

---

## 2. 消息类型系统

### 2.1 核心消息类型

消息类型的定义在 `src/types/message.js`（构建时生成），被 `src/utils/messages.ts` 大量引用。运行时 `Message` 是一个判别联合类型，核心分支：

| 类型 | 说明 | 参与 API | 参与链 |
|------|------|---------|--------|
| `user` | 用户输入、工具结果 (`tool_result`) | 是 | 是 |
| `assistant` | 模型回复（文本、thinking、tool_use） | 是 | 是 |
| `attachment` | 系统注入的附件（文件变更、权限上下文等） | 条件性 | 是 |
| `system` | 内部系统消息（compact_boundary 等） | 条件性 | 是 |
| `progress` | UI 进度条（Bash 输出、Sleep 倒计时） | 否 | **否** |

所有消息都带有 `uuid: UUID` 字段，用于去重和链构建。

```
Transcript 实际类型 = UserMessage | AssistantMessage | AttachmentMessage | SystemMessage
```

`src/utils/sessionStorage.ts` 中的 `isTranscriptMessage()` 是"什么算对话消息"的唯一判定标准：

```typescript
// src/utils/sessionStorage.ts:139-146
export function isTranscriptMessage(entry: Entry): entry is TranscriptMessage {
  return (
    entry.type === 'user' ||
    entry.type === 'assistant' ||
    entry.type === 'attachment' ||
    entry.type === 'system'
  )
}
```

### 2.2 序列化消息 (SerializedMessage)

写入磁盘时，消息被扩展为 `SerializedMessage`，附加会话级元数据：

```typescript
// src/types/logs.ts:8-17
export type SerializedMessage = Message & {
  cwd: string              // 当前工作目录
  userType: string         // 用户类型
  entrypoint?: string      // cli/sdk-ts/sdk-py 等入口标识
  sessionId: string        // 会话 ID
  timestamp: string        // ISO 时间戳
  version: string          // Claude Code 版本号
  gitBranch?: string       // Git 分支
  slug?: string            // 用于计划文件等的会话别名
}
```

### 2.3 转录消息 (TranscriptMessage)

`TranscriptMessage` 在 `SerializedMessage` 基础上增加了**对话树结构**字段：

```typescript
// src/types/logs.ts:221-231
export type TranscriptMessage = SerializedMessage & {
  parentUuid: UUID | null           // 父消息 UUID（链的核心）
  logicalParentUuid?: UUID | null   // 逻辑父消息（压缩时保留逻辑关系）
  isSidechain: boolean              // 是否为子 Agent 侧链
  gitBranch?: string
  agentId?: string                  // 子 Agent ID
  teamName?: string                 // Swarm 团队名
  agentName?: string                // Agent 自定义名称
  agentColor?: string               // Agent 颜色
  promptId?: string                 // OTel prompt.id 关联
}
```

**`parentUuid` 链的关键规则：**
- 普通消息：`parentUuid` 指向上一条链参与消息
- Compact Boundary 消息：`parentUuid = null`（截断链，标识压缩起点），`logicalParentUuid` 保留逻辑关系
- `progress` 类型不参与链（`isChainParticipant` 返回 `false`）

### 2.4 Entry 联合类型：JSONL 行的完整分类

JSONL 文件中每一行的类型由 `Entry` 联合定义，共 **20 种条目类型**：

```typescript
// src/types/logs.ts:297-317
export type Entry =
  | TranscriptMessage              // 用户/助手/附件/系统消息
  | SummaryMessage                 // 对话摘要
  | CustomTitleMessage             // 用户自定义标题
  | AiTitleMessage                 // AI 生成标题
  | LastPromptMessage              // 最后一个用户 prompt（用于 /resume 列表展示）
  | TaskSummaryMessage             // Agent 进度快照（每 5 步或 2 分钟更新）
  | TagMessage                     // 会话标签
  | AgentNameMessage               // Agent 名称
  | AgentColorMessage              // Agent 颜色
  | AgentSettingMessage            // Agent 配置
  | PRLinkMessage                  // 关联的 GitHub PR
  | FileHistorySnapshotMessage     // 文件修改历史快照
  | AttributionSnapshotMessage     // 代码归因快照
  | QueueOperationMessage          // 命令队列操作
  | SpeculationAcceptMessage       // 推测执行确认
  | ModeEntry                      // coordinator/normal 模式切换
  | WorktreeStateEntry             // Git worktree 状态
  | ContentReplacementEntry        // 大工具输出的溢写记录
  | ContextCollapseCommitEntry     // 上下文折叠提交
  | ContextCollapseSnapshotEntry   // 上下文折叠快照
```

各类型的写入逻辑详见 `appendEntry` 调度（[5.5节](#55-appendentry-调度逻辑)）。

### 2.5 消息过滤谓词

系统定义了明确的过滤函数来区分消息角色：

| 函数 | 用途 |
|------|------|
| `isTranscriptMessage()` | 判定是否为对话消息（user/assistant/attachment/system） |
| `isChainParticipant()` | 判定是否参与 parentUuid 链（排除 progress） |
| `isEphemeralToolProgress()` | 判定是否为临时进度条（bash_progress 等），不持久化 |
| `isCompactBoundaryMessage()` | 判定是否为压缩边界标记 |

---

## 3. 对话状态的内存管理

运行时对话存在两条并行管理路径，但共享同一个查询引擎。

### 3.1 REPL 路径：React 状态

交互式 CLI 中，消息数组存储在 React `useState` 中：

```typescript
// src/screens/REPL.tsx:1182
const [messages, rawSetMessages] = useState<MessageType[]>(initialMessages ?? []);
const messagesRef = useRef(messages);
```

为了避免 React 批处理导致的状态不一致，`setMessages` 被封装为**立即更新 ref + 延迟通知 React** 的模式（Zustand 模式）：

```typescript
// src/screens/REPL.tsx:1198-1201
const setMessages = useCallback((action: React.SetStateAction<MessageType[]>) => {
  const prev = messagesRef.current;
  const next = typeof action === 'function' ? action(messagesRef.current) : action;
  messagesRef.current = next;  // ref 立即更新，成为真相之源
  // ...React state 更新是渲染投影
```

消息变更会通过 `useLogMessages` hook 异步持久化到 JSONL。

### 3.2 SDK/Headless 路径：QueryEngine

`QueryEngine` 是一个独立类，拥有完整的对话生命周期管理能力：

```typescript
// src/QueryEngine.ts:184-207
export class QueryEngine {
  private config: QueryEngineConfig
  private mutableMessages: Message[]          // 可变消息数组
  private abortController: AbortController
  private permissionDenials: SDKPermissionDenial[]
  private totalUsage: NonNullableUsage
  private readFileState: FileStateCache
  private discoveredSkillNames = new Set<string>()
  private loadedNestedMemoryPaths = new Set<string>()

  constructor(config: QueryEngineConfig) {
    this.config = config
    this.mutableMessages = config.initialMessages ?? []
    // ...
  }
```

每次 `submitMessage()` 调用代表一个新 turn：

```typescript
// src/QueryEngine.ts:209-211
async *submitMessage(
  prompt: string | ContentBlockParam[],
  options?: { uuid?: string; isMeta?: boolean },
): AsyncGenerator<SDKMessage, void, unknown> {
```

**与 REPL 路径的关键区别**：

| 方面 | REPL | QueryEngine |
|------|------|-------------|
| 状态管理 | React useState + ref | 类成员 mutableMessages |
| 持久化触发 | useLogMessages hook（fire-and-forget） | 显式 await recordTranscript |
| 持久化时机 | 每次 setMessages | 进入 query 循环前 |
| 用户交互 | Ink 终端 UI | 无（纯异步生成器） |

### 3.3 两条路径的统一点

两条路径在以下层面完全统一：
- **查询引擎**：都调用 `query()` → `queryLoop()`
- **消息格式**：同一个 `Message` 类型系统
- **持久化格式**：同一个 `recordTranscript()` 函数
- **API 调用**：同一个 `callModel()` / `queryModelWithStreaming()`
- **工具执行**：同一套 `StreamingToolExecutor` / `runTools`

---

## 4. 核心查询循环 (queryLoop)

`queryLoop` 是整个 Agent 行为的核心驱动力，定义在 `src/query.ts`。

### 4.1 循环状态 (State)

每次循环迭代共享一个可变状态对象：

```typescript
// src/query.ts:204-217
type State = {
  messages: Message[]                    // 当前消息数组
  toolUseContext: ToolUseContext          // 工具使用上下文（权限、配置、abort 等）
  autoCompactTracking: AutoCompactTrackingState | undefined  // 压缩追踪
  maxOutputTokensRecoveryCount: number   // 输出 token 上限恢复计数
  hasAttemptedReactiveCompact: boolean   // 是否尝试过响应式压缩
  maxOutputTokensOverride: number | undefined
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined
  stopHookActive: boolean | undefined
  turnCount: number                      // 当前轮次计数
  transition: Continue | undefined       // 上一轮续行原因
}
```

循环入口：

```typescript
// src/query.ts:307
while (true) {
  let { toolUseContext } = state
  const { messages, autoCompactTracking, /* ... */ } = state
  // ... 完整的迭代逻辑
}
```

### 4.2 消息预处理管道

在每次 API 调用前，消息数组要经过一个**六层预处理管道**，从轻量到重量逐级压缩：

```
原始 messages[]
    │
    ▼ ① getMessagesAfterCompactBoundary()
    │   — 只取最后一次压缩边界之后的消息
    │
    ▼ ② applyToolResultBudget()
    │   — 大工具输出溢写到磁盘，留占位符
    │
    ▼ ③ snipCompactIfNeeded() [feature flag: HISTORY_SNIP]
    │   — 轻量级裁剪旧消息
    │
    ▼ ④ microcompact()
    │   — 微压缩：缩减工具结果中的冗余内容
    │
    ▼ ⑤ contextCollapse.applyCollapsesIfNeeded() [feature flag: CONTEXT_COLLAPSE]
    │   — 折叠不活跃的对话段落为摘要
    │
    ▼ ⑥ autocompact()
    │   — 全量压缩（token 超阈值时触发）
    │
    ▼ messagesForQuery — 发送给 API 的最终消息
```

对应的代码流程：

```typescript
// src/query.ts:365-467（简化）
let messagesForQuery = [...getMessagesAfterCompactBoundary(messages)]

messagesForQuery = await applyToolResultBudget(messagesForQuery, ...)

if (feature('HISTORY_SNIP')) {
  const snipResult = snipModule!.snipCompactIfNeeded(messagesForQuery)
  messagesForQuery = snipResult.messages
}

const microcompactResult = await deps.microcompact(messagesForQuery, ...)
messagesForQuery = microcompactResult.messages

if (feature('CONTEXT_COLLAPSE') && contextCollapse) {
  const collapseResult = await contextCollapse.applyCollapsesIfNeeded(messagesForQuery, ...)
  messagesForQuery = collapseResult.messages
}

const { compactionResult } = await deps.autocompact(messagesForQuery, ...)
if (compactionResult) {
  messagesForQuery = buildPostCompactMessages(compactionResult)
}
```

### 4.3 API 调用与流式响应

System Prompt 的最终构建：

```typescript
// src/query.ts:449-451
const fullSystemPrompt = asSystemPrompt(
  appendSystemContext(systemPrompt, systemContext),
)
```

API 流式调用通过 `deps.callModel()` 发起（生产实现为 `queryModelWithStreaming`）：

```typescript
// src/query.ts:659-670（简化）
for await (const message of deps.callModel({
  messages: prependUserContext(messagesForQuery, userContext),
  systemPrompt: fullSystemPrompt,
  thinkingConfig: toolUseContext.options.thinkingConfig,
  tools: toolUseContext.options.tools,
  signal: toolUseContext.abortController.signal,
  options: { model: currentModel, /* ... */ },
})) {
  // 处理每个流式事件
}
```

`prependUserContext` 在消息数组最前面插入一条携带 `<system-reminder>` 的 `UserMessage`，注入 CLAUDE.md 等上下文。

在 API 层 (`src/services/api/claude.ts`)，消息还要经过最终的正规化：
- `normalizeMessagesForAPI()` — 重排附件、合并连续 user/assistant、丢弃虚拟消息
- `stripToolReferenceBlocksFromUserMessage()` — 移除 tool reference（非 tool-search 模式）
- `ensureToolResultPairing()` — 确保 tool_use/tool_result 配对
- `stripExcessMediaItems()` — 限制媒体数量
- `userMessageToMessageParam()` / `assistantMessageToMessageParam()` — 转为 Anthropic SDK 格式

### 4.4 工具执行与结果注入

工具执行有两种路径：

**流中执行（Streaming Tool Executor）**：

```typescript
// src/query.ts:561-568
let streamingToolExecutor = useStreamingToolExecution
  ? new StreamingToolExecutor(
      toolUseContext.options.tools,
      canUseTool,
      toolUseContext,
    )
  : null
```

工具输入在流式过程中逐步填充，完整后立即开始执行，不等待模型完成全部输出。

**批量执行（runTools）**：

流结束后对所有 tool_use 块执行，用于流式执行被禁用的场景。

工具结果以 `UserMessage`（携带 `tool_result` content block）形式注入，在下一轮迭代中与 assistant 消息一起构成新的消息数组。

### 4.5 循环续行与终止条件

**续行（下一轮迭代）**：当存在未完成的工具调用时：

```typescript
// src/query.ts:1714-1728
const next: State = {
  messages: [...messagesForQuery, ...assistantMessages, ...toolResults],
  toolUseContext: toolUseContextWithQueryTracking,
  autoCompactTracking: tracking,
  turnCount: nextTurnCount,
  maxOutputTokensRecoveryCount: 0,
  hasAttemptedReactiveCompact: false,
  pendingToolUseSummary: nextPendingToolUseSummary,
  maxOutputTokensOverride: undefined,
  stopHookActive,
  transition: { reason: 'next_turn' },
}
state = next
```

**终止条件**：

| 条件 | 返回值 |
|------|--------|
| 模型未调用工具且 stop_reason != tool_use | `{ reason: 'end_turn' }` |
| 达到 maxTurns 限制 | `{ reason: 'max_turns' }` |
| Token 超过阻塞限制 | `{ reason: 'blocking_limit' }` |
| 用户中断（abort signal） | 抛出错误 |
| Token 预算耗尽 | `{ reason: 'budget_exhausted' }` |

```typescript
// src/query.ts:1704-1711
if (maxTurns && nextTurnCount > maxTurns) {
  yield createAttachmentMessage({
    type: 'max_turns_reached',
    maxTurns,
    turnCount: nextTurnCount,
  })
  return { reason: 'max_turns', turnCount: nextTurnCount }
}
```

### 4.6 完整消息流转路径

```
用户键入 "请修复这个 bug"
    │
    ▼ processUserInput()
    │  → 构建 UserMessage { type: 'user', content: [...], uuid: 'abc-123' }
    │  → 可能附带 AttachmentMessage (文件上下文、权限声明等)
    │
    ▼ 追加到 messages 数组 (REPL: setMessages / SDK: mutableMessages.push)
    │
    ▼ recordTranscript() — 持久化到 JSONL（SDK 路径在此 await）
    │
    ▼ query({ messages, systemPrompt, userContext, ... })
    │
    ▼ queryLoop 迭代 1:
    │  ├── 预处理管道 → messagesForQuery
    │  ├── callModel({ messages: prependUserContext(messagesForQuery, ...) })
    │  │   ├── [stream event] content_block: text "我来看看这个文件..."
    │  │   ├── [stream event] content_block: tool_use { name: 'Read', input: { path: '/src/bug.ts' } }
    │  │   └── [message_delta] stop_reason: 'tool_use'
    │  │
    │  ├── assistantMessages = [AssistantMessage(text), AssistantMessage(tool_use)]
    │  ├── StreamingToolExecutor.execute(Read, { path: '/src/bug.ts' })
    │  │   → toolResults = [UserMessage(tool_result: 文件内容)]
    │  │
    │  └── state = { messages: [...messagesForQuery, ...assistant, ...toolResults], turnCount: 2 }
    │
    ▼ queryLoop 迭代 2:
    │  ├── callModel(...) → text "找到了问题..." + tool_use(Edit)
    │  ├── Execute Edit tool → toolResults
    │  └── state.turnCount = 3
    │
    ▼ queryLoop 迭代 3:
    │  ├── callModel(...) → text "已修复，修改如下..." (无 tool_use)
    │  └── return { reason: 'end_turn' }  ← 循环结束
    │
    ▼ yield 所有 messages 给调用方（REPL 渲染 / SDK 返回）
```

---

## 5. 本地持久化：JSONL 存储

### 5.1 文件结构与路径

```
~/.claude/                              # CLAUDE_CONFIG_DIR 可覆盖
├── history.jsonl                       # 提示历史（↑键、Ctrl+R），非对话记录
├── projects/
│   └── <sanitized-project-path>/       # 按项目路径分目录
│       ├── <session-uuid>.jsonl        # 主会话转录文件
│       ├── <session-uuid>/             # 会话子目录
│       │   ├── subagents/
│       │   │   ├── agent-<agentId>.jsonl    # 子 Agent 侧链
│       │   │   └── workflows/
│       │   │       └── <runId>/
│       │   │           └── agent-<agentId>.jsonl
│       │   ├── tool-results/           # 溢写的大工具输出
│       │   └── summary.md              # Session Memory
│       └── memory/
│           └── MEMORY.md               # Auto Memory
```

路径生成：

```typescript
// src/utils/sessionStorage.ts:202-204
export function getTranscriptPath(): string {
  const projectDir = getSessionProjectDir() ?? getProjectDir(getOriginalCwd())
  return join(projectDir, `${getSessionId()}.jsonl`)
}
```

**体积限制**：JSONL 文件可能增长到数 GB。读取时有 50MB 的安全阈值：

```typescript
// src/utils/sessionStorage.ts:229
export const MAX_TRANSCRIPT_READ_BYTES = 50 * 1024 * 1024
```

### 5.2 Project 类：写入引擎

`Project` 是对话存储的核心管理类，维护会话级元数据缓存和写入队列：

```typescript
// src/utils/sessionStorage.ts:532-568（简化）
class Project {
  currentSessionTag: string | undefined
  currentSessionTitle: string | undefined
  currentSessionAgentName: string | undefined
  currentSessionLastPrompt: string | undefined
  currentSessionMode: 'coordinator' | 'normal' | undefined
  currentSessionWorktree: PersistedWorktreeSession | null | undefined

  sessionFile: string | null = null
  private pendingEntries: Entry[] = []           // 未物化前的缓冲区
  private remoteIngressUrl: string | null = null // 远程同步 URL
  private internalEventWriter: InternalEventWriter | null = null  // CCR v2
  private writeQueues = new Map<string, Array<{ entry: Entry; resolve: () => void }>>()
  private FLUSH_INTERVAL_MS = 100                // 刷新间隔
  private readonly MAX_CHUNK_BYTES = 100 * 1024 * 1024  // 单次写入上限 100MB
}
```

### 5.3 写入队列与批量刷新

写入采用**队列 + 定时刷新**模式，避免频繁磁盘 IO：

```
enqueueWrite(filePath, entry)
    → 放入 writeQueues[filePath]
    → scheduleDrain()
        → 100ms 后触发 drainWriteQueue()
            → 对每个文件，批量拼接 JSON 行
            → 超过 100MB 时分块写入
            → appendToFile(filePath, content)
```

```typescript
// src/utils/sessionStorage.ts:645-678
private async drainWriteQueue(): Promise<void> {
  for (const [filePath, queue] of this.writeQueues) {
    const batch = queue.splice(0)
    let content = ''
    const resolvers: Array<() => void> = []

    for (const { entry, resolve } of batch) {
      const line = jsonStringify(entry) + '\n'
      if (content.length + line.length >= this.MAX_CHUNK_BYTES) {
        await this.appendToFile(filePath, content)  // 分块刷盘
        for (const r of resolvers) r()
        resolvers.length = 0
        content = ''
      }
      content += line
      resolvers.push(resolve)
    }
    if (content.length > 0) {
      await this.appendToFile(filePath, content)
      for (const r of resolvers) r()
    }
  }
}
```

### 5.4 延迟物化：避免空文件

Session 文件**不在启动时创建**，而是在第一条 user/assistant 消息时才物化：

```typescript
// src/utils/sessionStorage.ts:976-991
private async materializeSessionFile(): Promise<void> {
  if (this.shouldSkipPersistence()) return
  this.ensureCurrentSessionFile()
  this.reAppendSessionMetadata()  // 写入标题、标签等元数据
  if (this.pendingEntries.length > 0) {
    const buffered = this.pendingEntries
    this.pendingEntries = []
    for (const entry of buffered) {
      await this.appendEntry(entry)
    }
  }
}
```

物化前的条目（如 `mode`、`agent-setting`）暂存在 `pendingEntries` 数组中。

### 5.5 appendEntry 调度逻辑

`appendEntry` 是所有条目写入的入口，按类型分发处理：

```
appendEntry(entry)
    │
    ├── 持久化已禁用? → return
    │
    ├── sessionFile === null?
    │   → 缓冲到 pendingEntries → return
    │
    ├── entry.type === 'summary' / 'custom-title' / 'ai-title' / 'last-prompt'
    │   / 'task-summary' / 'tag' / 'agent-name' / 'agent-color' / 'agent-setting'
    │   / 'pr-link' / 'file-history-snapshot' / 'attribution-snapshot'
    │   / 'speculation-accept' / 'mode' / 'worktree-state'
    │   → 直接 enqueueWrite（无去重）
    │
    ├── entry.type === 'content-replacement'
    │   → 有 agentId? → 写到 agent 侧链文件
    │   → 否则 → 写到主会话文件
    │
    ├── entry.type === 'marble-origami-commit' / 'marble-origami-snapshot'
    │   → 直接写入（顺序敏感）
    │
    ├── entry.type === 'queue-operation'
    │   → 直接写入
    │
    └── TranscriptMessage (user/assistant/attachment/system)
        → 检查 UUID 去重（避免压缩/重放时重复写入）
        → 侧链消息 → 写到 agent-<agentId>.jsonl（跳过主文件去重）
        → 主链消息 → 写到主会话 JSONL + persistToRemote()
```

### 5.6 insertMessageChain：消息链写入

`insertMessageChain` 是将一批消息写入 JSONL 的核心方法，负责维护 `parentUuid` 链：

```typescript
// src/utils/sessionStorage.ts:993-1069（简化）
async insertMessageChain(
  messages: Transcript,
  isSidechain: boolean = false,
  agentId?: string,
  startingParentUuid?: UUID | null,
  teamInfo?: { teamName?: string; agentName?: string },
) {
  let parentUuid: UUID | null = startingParentUuid ?? null

  // 第一条 user/assistant 消息触发文件物化
  if (this.sessionFile === null && messages.some(m => m.type === 'user' || m.type === 'assistant')) {
    await this.materializeSessionFile()
  }

  const gitBranch = await getBranch()
  const slug = getPlanSlugCache().get(getSessionId())

  for (const message of messages) {
    // tool_result 消息使用对应 assistant 消息的 UUID 作为 parent
    let effectiveParentUuid = parentUuid
    if (message.type === 'user' && message.sourceToolAssistantUUID) {
      effectiveParentUuid = message.sourceToolAssistantUUID
    }

    const transcriptMessage: TranscriptMessage = {
      parentUuid: isCompactBoundary ? null : effectiveParentUuid,
      logicalParentUuid: isCompactBoundary ? parentUuid : undefined,
      isSidechain,
      teamName, agentName, agentId, promptId,
      ...message,
      // 会话字段必须在 spread 之后，防止 resume/fork 时被源消息覆盖
      userType: getUserType(),
      cwd: getCwd(),
      sessionId,
      version: VERSION,
      gitBranch, slug,
    }
    await this.appendEntry(transcriptMessage)

    if (isChainParticipant(message)) {
      parentUuid = message.uuid  // 推进链
    }
  }
}
```

### 5.7 增量写入 Hook (useLogMessages)

REPL 侧通过 React hook 实现 **fire-and-forget** 式增量持久化：

```typescript
// src/hooks/useLogMessages.ts:19-69（简化）
export function useLogMessages(messages: Message[], ignore: boolean = false) {
  const lastRecordedLengthRef = useRef(0)
  const lastParentUuidRef = useRef<UUID | undefined>(undefined)
  const firstMessageUuidRef = useRef<UUID | undefined>(undefined)

  useEffect(() => {
    if (ignore) return

    // 检测是增量追加还是压缩重建
    const isIncremental =
      currentFirstUuid === firstMessageUuidRef.current &&
      prevLength <= messages.length

    // 增量：只取新增部分；压缩后：传全量（recordTranscript 内部去重）
    const slice = startIndex === 0 ? messages : messages.slice(startIndex)
    const parentHint = isIncremental ? lastParentUuidRef.current : undefined

    void recordTranscript(slice, teamInfo, parentHint, messages)
      .then(lastRecordedUuid => {
        // 更新追踪 ref
      })
  }, [messages])
}
```

**关键设计**：
- 增量追踪避免 O(n) 扫描：只传新增的尾部给 `recordTranscript`
- 压缩检测：通过 `firstMessageUuid` 变化判断是否发生了数组重建
- 不阻塞 UI：使用 `void` 调用，不 await 写入结果

### 5.8 recordTranscript：去重与链维护

```typescript
// src/utils/sessionStorage.ts:1408-1449
export async function recordTranscript(
  messages: Message[],
  teamInfo?: TeamInfo,
  startingParentUuidHint?: UUID,
  allMessages?: readonly Message[],
): Promise<UUID | null> {
  const cleanedMessages = cleanMessagesForLogging(messages, allMessages)
  const sessionId = getSessionId() as UUID
  const messageSet = await getSessionMessages(sessionId)  // 已写入 UUID 集合
  const newMessages: typeof cleanedMessages = []
  let startingParentUuid: UUID | undefined = startingParentUuidHint
  let seenNewMessage = false

  for (const m of cleanedMessages) {
    if (messageSet.has(m.uuid as UUID)) {
      // 已存在的消息只在构成前缀时追踪为 parent
      if (!seenNewMessage && isChainParticipant(m)) {
        startingParentUuid = m.uuid as UUID
      }
    } else {
      newMessages.push(m)
      seenNewMessage = true
    }
  }

  if (newMessages.length > 0) {
    await getProject().insertMessageChain(
      newMessages, false, undefined, startingParentUuid, teamInfo,
    )
  }

  return (lastRecorded?.uuid as UUID) ?? startingParentUuid ?? null
}
```

**前缀追踪规则**：已记录的消息只有在所有新消息**之前**出现时才被追踪为 parent。这确保了：
- 普通追加：已记录消息是前缀 → 正确的 parent 链
- 压缩后：新的 compact boundary 出现在前面 → 不追踪 messagesToKeep → boundary 的 `parentUuid = null`（正确截断链）

### 5.9 禁用持久化的条件

```typescript
// src/utils/sessionStorage.ts:960-969
private shouldSkipPersistence(): boolean {
  return (
    (getNodeEnv() === 'test' && !allowTestPersistence) ||
    getSettings_DEPRECATED()?.cleanupPeriodDays === 0 ||
    isSessionPersistenceDisabled() ||
    isEnvTruthy(process.env.CLAUDE_CODE_SKIP_PROMPT_HISTORY)
  )
}
```

---

## 6. 云端存储：远程持久化

**本地 JSONL 是所有场景下的真相之源。远程存储是可选的实时镜像。**

### 6.1 Session Ingress v1

当 `ENABLE_SESSION_PERSISTENCE` 环境变量开启且配置了 `remoteIngressUrl` 时，每条 `TranscriptMessage` 在写入本地的同时通过 HTTP PUT 同步到远端：

```typescript
// src/utils/sessionStorage.ts:1325-1343
// v1 Session Ingress path
if (
  !isEnvTruthy(process.env.ENABLE_SESSION_PERSISTENCE) ||
  !this.remoteIngressUrl
) {
  return  // 未启用则跳过
}

const success = await sessionIngress.appendSessionLog(
  sessionId,
  entry,
  this.remoteIngressUrl,
)

if (!success) {
  logEvent('tengu_session_persistence_failed', {})
  gracefulShutdownSync(1, 'other')  // 同步失败 → 优雅关闭
}
```

实现层 (`src/services/api/sessionIngress.ts`) 使用 `axios.put()` 发送单条条目，通过 `Last-Uuid` header 维护链的连续性。

**关键行为**：同步失败触发进程关闭，说明在启用远程持久化的部署模式中，数据一致性被视为**硬约束**。

### 6.2 CCR v2 内部事件通道

更新的架构使用 `internalEventWriter` 替代 HTTP PUT，支持更丰富的元数据：

```typescript
// src/utils/sessionStorage.ts:1307-1323
if (this.internalEventWriter) {
  try {
    await this.internalEventWriter(
      'transcript',
      entry as unknown as Record<string, unknown>,
      {
        ...(isCompactBoundaryMessage(entry) && { isCompaction: true }),
        ...(entry.agentId && { agentId: entry.agentId }),
      },
    )
  } catch {
    logEvent('tengu_session_persistence_failed', {})
    logForDebugging('Failed to write transcript as internal event')
  }
  return
}
```

v2 的失败处理更宽容（不关闭进程），且能标记压缩事件以便服务端优化存储。

### 6.3 远端恢复 (Hydration)

#### v1 Hydration

从远端拉取完整日志，**覆盖本地文件**：

```typescript
// src/utils/sessionStorage.ts:1587-1621
export async function hydrateRemoteSession(
  sessionId: string,
  ingressUrl: string,
): Promise<boolean> {
  switchSession(asSessionId(sessionId))
  const remoteLogs = (await sessionIngress.getSessionLogs(sessionId, ingressUrl)) || []

  const sessionFile = getTranscriptPathForSession(sessionId)
  const content = remoteLogs.map(e => jsonStringify(e) + '\n').join('')
  await writeFile(sessionFile, content, { encoding: 'utf8', mode: 0o600 })

  // 写入完成后才启用远程持久化，确保先同步再写入
  project.setRemoteIngressUrl(ingressUrl)
  return remoteLogs.length > 0
}
```

#### CCR v2 Hydration

更复杂，需要处理主线程和多个子 Agent 的事件流：

```typescript
// src/utils/sessionStorage.ts:1632-1703（简化）
export async function hydrateFromCCRv2InternalEvents(sessionId: string): Promise<boolean> {
  // 1. 拉取主线程事件 → 写入主会话 JSONL
  const events = await reader()
  const fgContent = events.map(e => jsonStringify(e.payload) + '\n').join('')
  await writeFile(sessionFile, fgContent, ...)

  // 2. 拉取子 Agent 事件 → 按 agentId 分组 → 写入各自 JSONL
  const subagentEvents = await subagentReader()
  const byAgent = new Map<string, Record<string, unknown>[]>()
  for (const e of subagentEvents) {
    byAgent.get(e.agent_id)?.push(e.payload) // 按 agentId 分桶
  }
  for (const [agentId, entries] of byAgent) {
    await writeFile(getAgentTranscriptPath(agentId), entries.map(...).join(''), ...)
  }
}
```

服务端在返回数据时已处理压缩过滤——只返回最后一次压缩边界之后的事件。

### 6.4 本地与云端的关系

```
写入路径：
  消息 ──┬──→ 本地 JSONL（始终写入）
         │
         └──→ 远端（可选双写）
               ├─ v1: HTTP PUT → Session Ingress 服务
               └─ v2: internalEventWriter → CCR 事件总线

恢复路径：
  远端 ──hydrateRemoteSession()──→ 覆盖本地 JSONL ──→ 从本地加载
                                                        │
                                                        ▼
                                                   buildConversationChain()
                                                   deserializeMessages()
                                                   → 恢复内存中的 Message[]
```

| 对比项 | 本地 JSONL | 远端存储 |
|--------|-----------|---------|
| 存在条件 | 始终（除非显式禁用） | 需要 ENABLE_SESSION_PERSISTENCE 或 CCR v2 |
| 写入方式 | append-only 文件追加 | HTTP PUT / 事件写入 |
| 读取方式 | 直接读文件 | getSessionLogs API |
| 真相地位 | **主** — 恢复后所有读取从本地 | **副** — 恢复时覆盖本地，然后退居幕后 |
| 体积管理 | 文件可增长至数 GB | 服务端可做压缩过滤 |

---

## 7. 对话恢复 (Resume)

### 7.1 日志加载与链重建

恢复对话的核心流程在 `src/utils/conversationRecovery.ts` 和 `src/utils/sessionStorage.ts`：

```
/resume 或 --continue
    │
    ▼ loadMessageLogs() / getLastSessionLog()
    │  → 列出项目目录下的所有 .jsonl 文件
    │  → 提取 LogOption 元数据（标题、时间、首条 prompt 等）
    │
    ▼ loadTranscriptFile(path)
    │  → 逐行解析 JSONL
    │  → 按类型分类（TranscriptMessage / metadata / 其他）
    │  → 处理遗留 progress 条目的链桥接
    │
    ▼ buildConversationChain(entries, leafUuid?)
    │  → 从最新的非侧链叶节点开始
    │  → 沿 parentUuid 链向上回溯
    │  → 反转 → 得到时间序列的消息数组
    │
    ▼ removeExtraFields()
    │  → 去除 parentUuid、isSidechain 等持久化专用字段
    │  → 还原为纯 Message 格式
    │
    ▼ deserializeMessages() → 恢复后的 Message[]
```

### 7.2 消息反序列化

`deserializeMessages` 不是简单的 JSON 解析，而是一个**清洗+修复**管道：

```typescript
// src/utils/conversationRecovery.ts:154-252（简化）
export function deserializeMessages(serializedMessages: Message[]): Message[] {
  // 1. 迁移遗留附件类型
  const migratedMessages = serializedMessages.map(migrateLegacyAttachmentTypes)

  // 2. 清除无效的 permissionMode 值
  for (const msg of migratedMessages) {
    if (msg.type === 'user' && !validModes.has(msg.permissionMode)) {
      msg.permissionMode = undefined
    }
  }

  // 3. 过滤未解析的工具调用（助手发了 tool_use 但没有对应 tool_result）
  const filteredToolUses = filterUnresolvedToolUses(migratedMessages)

  // 4. 过滤孤立的纯思考助手消息（可能导致 API 错误）
  const filteredThinking = filterOrphanedThinkingOnlyMessages(filteredToolUses)

  // 5. 过滤仅含空白的助手消息
  const filteredMessages = filterWhitespaceOnlyAssistantMessages(filteredThinking)

  // 6. 如果最后一条是 user 消息，追加合成的 assistant 占位符
  if (lastMessage.type === 'user') {
    filteredMessages.splice(idx + 1, 0, createAssistantMessage({
      content: NO_RESPONSE_REQUESTED,
    }))
  }

  return filteredMessages
}
```

### 7.3 中断检测与自动续行

系统能检测对话是在哪个阶段被中断的：

```typescript
// src/utils/conversationRecovery.ts:272-333
function detectTurnInterruption(messages): InternalInterruptionState {
  // 找到最后一条非 system/progress 的消息
  const lastMessage = messages.findLast(m => m.type !== 'system' && m.type !== 'progress')

  if (lastMessage.type === 'assistant') return { kind: 'none' }  // 正常结束
  if (lastMessage.type === 'user') {
    if (lastMessage.isMeta) return { kind: 'none' }
    if (isToolUseResultMessage(lastMessage)) {
      // 检查是否为终端工具（如 SendUserMessage），如果是则为正常结束
      if (isTerminalToolResult(lastMessage, messages, idx)) return { kind: 'none' }
      return { kind: 'interrupted_turn' }     // 工具执行中断
    }
    return { kind: 'interrupted_prompt', message: lastMessage }  // 用户输入后中断
  }
  if (lastMessage.type === 'attachment') return { kind: 'interrupted_turn' }
}
```

对于 `interrupted_turn`（工具执行中断），自动注入续行消息：

```typescript
const continuationMessage = createUserMessage({
  content: 'Continue from where you left off.',
  isMeta: true,
})
filteredMessages.push(continuationMessage)
```

### 7.4 附属状态恢复

对话恢复不仅仅是消息——`src/utils/sessionRestore.ts` 负责恢复完整的会话状态：

- **文件修改历史** (`FileHistorySnapshot`) → `copyFileHistoryForResume()`
- **代码归因信息** (`AttributionSnapshotMessage`)
- **上下文折叠存储** (`ContextCollapseCommitEntry` + `ContextCollapseSnapshotEntry`)
- **技能状态** → `restoreSkillStateFromMessages()`
- **计划文件** → `copyPlanForResume()`
- **Worktree 状态** → 检查 worktree 目录是否仍存在

---

## 8. 上下文窗口管理

### 8.1 模型上下文窗口与 token 上限

```typescript
// src/utils/context.ts
export const MODEL_CONTEXT_WINDOW_DEFAULT = 200_000
export const CAPPED_DEFAULT_MAX_TOKENS = 8_000       // 默认最大输出 token
export const ESCALATED_MAX_TOKENS = 64_000            // 升级后的最大输出 token

export function getContextWindowForModel(model: string, betas?: string[]): number {
  if (has1mContext(model)) return 1_000_000
  // ...
}
```

### 8.2 自动压缩 (Auto Compact)

有效上下文窗口 = 模型窗口 - 摘要输出预留空间：

```typescript
// src/services/compact/autoCompact.ts:32-49
export function getEffectiveContextWindowSize(model: string): number {
  const reservedTokensForSummary = Math.min(
    getMaxOutputTokensForModel(model),
    MAX_OUTPUT_TOKENS_FOR_SUMMARY,  // 20,000
  )
  let contextWindow = getContextWindowForModel(model, getSdkBetas())
  // 可通过环境变量覆盖
  if (process.env.CLAUDE_CODE_AUTO_COMPACT_WINDOW) {
    contextWindow = Math.min(contextWindow, parsed)
  }
  return contextWindow - reservedTokensForSummary
}
```

触发阈值 = 有效窗口 - 缓冲区：

```typescript
// src/services/compact/autoCompact.ts:62-77
export const AUTOCOMPACT_BUFFER_TOKENS = 13_000
export const WARNING_THRESHOLD_BUFFER_TOKENS = 20_000

export function getAutoCompactThreshold(model: string): number {
  return getEffectiveContextWindowSize(model) - AUTOCOMPACT_BUFFER_TOKENS
}
```

**熔断机制**：连续压缩失败超过 3 次停止重试：

```typescript
const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3
```

### 8.3 工具结果预算 (Tool Result Budget)

过大的工具输出（如读取大文件）不会全部放入消息数组，而是溢写到磁盘：

```
applyToolResultBudget(messages, contentReplacementState, ...)
    │
    ├── 扫描所有 tool_result 内容块
    ├── 超过预算的 → 写入 tool-results/ 目录
    ├── 消息中替换为占位符引用
    └── 记录替换决策到 ContentReplacementEntry（供 resume 重放）
```

### 8.4 阻塞限制与预防性拒绝

当自动压缩被禁用时，系统在 token 超过硬限制时**拒绝发送 API 请求**：

```typescript
// src/query.ts:628-648（简化）
if (!compactionResult && !reactiveCompact?.isEnabled() && !collapseOwnsIt) {
  const { isAtBlockingLimit } = calculateTokenWarningState(
    tokenCountWithEstimation(messagesForQuery) - snipTokensFreed,
    toolUseContext.options.mainLoopModel,
  )
  if (isAtBlockingLimit) {
    yield createAssistantAPIErrorMessage({
      content: PROMPT_TOO_LONG_ERROR_MESSAGE,
    })
    return { reason: 'blocking_limit' }
  }
}
```

### 8.5 Token 预算追踪

可选的 USD/token 总预算机制：

```typescript
// src/query.ts:280
const budgetTracker = feature('TOKEN_BUDGET') ? createBudgetTracker() : null
```

`task_budget.remaining` 跨压缩边界维护，确保预算计算考虑被压缩掉的 token 消耗。

---

## 9. 子 Agent 对话管理

子 Agent（通过 `AgentTool` 或 Swarm 创建）有独立但关联的对话管理：

| 方面 | 主 Agent | 子 Agent |
|------|---------|---------|
| 消息存储 | `<sessionId>.jsonl` | `<sessionId>/subagents/agent-<agentId>.jsonl` |
| parentUuid 链 | 主链 | 独立侧链（`isSidechain: true`） |
| UUID 去重 | 主文件集合 | **跳过主文件集合去重**（fork 继承的消息共享 UUID） |
| 远程同步 | 主链写入远端 | 通过 `agentId` 路由到子 Agent 事件流 |
| 恢复 | 主链恢复 | 通过 `agentId` 读取对应侧链文件 |

侧链的 UUID 去重跳过设计有明确的工程原因：

```typescript
// src/utils/sessionStorage.ts:1230-1256
// Skip dedup for agent sidechain LOCAL writes — they go to a separate
// file, and fork-inherited parent messages share UUIDs with the main
// session transcript. Deduping against the main session's set would
// drop them, leaving the persisted sidechain transcript incomplete.
//
// The sidechain bypass applies ONLY to the local file write — remote
// persistence uses a single Last-Uuid chain per sessionId, so
// re-POSTing a UUID it already has 409s and eventually exhausts retries.
```

---

## 10. 其他数据通道

除了对话 JSONL 和 API 请求，还有以下数据通道：

| 通道 | 目标 | 内容 | 触发条件 |
|------|------|------|---------|
| **1P Analytics** | `api.anthropic.com/api/event_logging/batch` | 事件名 + 数值/布尔元数据（**不含字符串/代码**） | 始终（第一方部署） |
| **Session Ingress** | 配置的 ingress URL | 完整 TranscriptMessage | `ENABLE_SESSION_PERSISTENCE` |
| **CCR v2 事件** | 内部事件总线 | 完整转录条目 | CCR 部署 |
| **用户反馈分享** | `api.anthropic.com/api/claude_code_shared_session_transcripts` | **脱敏后的**对话 + 可选子 Agent 转录 | 用户主动提交 |
| **OTel** | 可配置的 exporters | session/version/account 属性 | 配置启用 |
| **Datadog** | Datadog API | 白名单事件 | 一方部署 + 白名单 |
| **Teleport** | `/v1/sessions/{id}/events` | 用户事件 | 远程环境 |
| **提示历史** | `~/.claude/history.jsonl` | 用户输入的 prompt 文本 | 每次输入 |

---

## 11. 关键源文件索引

| 职责 | 文件路径 | 核心导出 |
|------|----------|---------|
| 消息类型定义 | `src/types/message.js` (generated) | `Message`, `UserMessage`, `AssistantMessage` 等 |
| 持久化类型 | `src/types/logs.ts` | `SerializedMessage`, `TranscriptMessage`, `Entry`, `LogOption` |
| 存储引擎 | `src/utils/sessionStorage.ts` | `Project`, `recordTranscript`, `loadTranscriptFile`, `buildConversationChain`, `hydrateRemoteSession` |
| 消息工具函数 | `src/utils/messages.ts` | `normalizeMessagesForAPI`, `getMessagesAfterCompactBoundary`, `filterUnresolvedToolUses` |
| 核心查询循环 | `src/query.ts` | `query()`, `queryLoop()`, `State` |
| SDK 对话引擎 | `src/QueryEngine.ts` | `QueryEngine`, `submitMessage()` |
| REPL 消息状态 | `src/screens/REPL.tsx` | `useState<Message[]>`, `setMessages` |
| 增量日志 Hook | `src/hooks/useLogMessages.ts` | `useLogMessages()` |
| 对话恢复 | `src/utils/conversationRecovery.ts` | `deserializeMessages`, `detectTurnInterruption`, `restoreSkillStateFromMessages` |
| 附属状态恢复 | `src/utils/sessionRestore.ts` | 文件历史、归因、上下文折叠、TODO 等恢复 |
| System Prompt | `src/utils/systemPrompt.ts` | `buildEffectiveSystemPrompt` |
| 用户上下文注入 | `src/utils/api.ts` | `prependUserContext`, `appendSystemContext` |
| API 调用与流式 | `src/services/api/claude.ts` | `queryModel`, `queryModelWithStreaming`, `normalizeMessagesForAPI` |
| 自动压缩 | `src/services/compact/autoCompact.ts` | `getAutoCompactThreshold`, `getEffectiveContextWindowSize` |
| 压缩执行 | `src/services/compact/compact.ts` | `compactConversation` |
| 上下文窗口 | `src/utils/context.ts` | `getContextWindowForModel`, `MODEL_CONTEXT_WINDOW_DEFAULT` |
| Token 计数 | `src/utils/tokens.ts` | `tokenCountWithEstimation`, `finalContextTokensFromLastResponse` |
| 工具结果溢写 | `src/utils/toolResultStorage.ts` | `applyToolResultBudget`, `recordContentReplacement` |
| Session Ingress | `src/services/api/sessionIngress.ts` | `appendSessionLog`, `getSessionLogs` |
| 会话全局状态 | `src/bootstrap/state.ts` | `sessionId`, `parentSessionId` |
| 清除对话 | `src/commands/clear/conversation.ts` | 清除会话、重置缓存 |
| 提示历史 | `src/history.ts` | `~/.claude/history.jsonl` 管理 |
