---
title: SDK / Headless 模式
aliases: [SDK 系统, SDK System, Headless 模式]
series: Claude Code 架构解析
category: 运行模式
order: 16
tags:
  - claude-code
  - sdk
  - headless
  - ndjson
  - rpc
date: 2026-04-04
---

# SDK / Headless 模式详细技术文档

## 1. 概述

Claude Code 不仅是一个交互式 CLI 工具，它还可以作为 **SDK（Software Development Kit）** 被其他程序以编程方式调用。SDK 模式也称为 **Headless 模式**——没有终端 UI，通过结构化的 NDJSON 协议进行双向通信。

这使得 Claude Code 能够被嵌入到：
- **IDE 插件**（VS Code、JetBrains）
- **claude.ai 网页版**（通过 CCR/Bridge）
- **守护进程**（Daemon）用于后台任务调度
- **第三方应用程序**通过 Node.js SDK 集成

### 核心设计哲学

SDK 模式没有独立的入口文件。它**复用 CLI 的同一条启动路径**，只在 `--print / -p` 分支中分叉为无 UI 的 Headless 执行。这种设计避免了双代码路径的维护负担，确保交互式和编程式使用共享完全相同的 QueryEngine 和工具链。

---

## 2. 架构总览

```
┌──────────────────────────────────────────────────────────────────┐
│                     SDK 消费者 (VS Code / claude.ai / Daemon)      │
│                                                                   │
│  stdin (NDJSON)                                  stdout (NDJSON)  │
│  ─────────────►                                ◄─────────────── │
│  SDKUserMessage                                SDKMessage          │
│  control_response                              control_request     │
│  keep_alive                                    assistant/result    │
│  update_env_vars                               stream_event        │
│                                                system events       │
└───────────┬──────────────────────────────────────┬────────────────┘
            │                                      │
            ▼                                      ▼
┌──────────────────────────────────────────────────────────────────┐
│                        StructuredIO                               │
│                                                                   │
│  structuredInput ◄── read() async generator ◄── stdin NDJSON      │
│                                                                   │
│  outbound ──────► Stream<StdoutMessage> ──────► write() ──► stdout│
│                                                                   │
│  pendingRequests Map ── control_request/response 配对追踪          │
│  createCanUseTool() ── 权限决策: 规则引擎 + Hook + SDK prompt 竞速 │
│  sendRequest<T>() ── 发送 control_request 并等待 response          │
│  handleElicitation() ── MCP elicitation 委托给 SDK 消费者          │
│  sendMcpMessage() ── MCP JSON-RPC 桥接                            │
└───────────┬──────────────────────────────────────┬────────────────┘
            │                                      │
            ▼                                      ▼
┌────────────────────────┐     ┌────────────────────────────────────┐
│    runHeadlessStreaming │     │           QueryEngine              │
│    (print.ts)          │     │                                    │
│                        │     │  ask() → submitMessage()           │
│  stdin loop:           │────►│  messages → LLM → tools → yield   │
│  - user messages       │     │  yield SDKMessage back             │
│  - control_request     │◄────│                                    │
│  - initialize          │     │  setSDKStatus() → output.enqueue  │
│  - interrupt           │     │  includePartialMessages → stream   │
│  - set_model/mode      │     └────────────────────────────────────┘
│  - mcp_*               │
│  - hook_callback       │     ┌────────────────────────────────────┐
│  - rewind_files        │     │         sdkEventQueue              │
│  - seed_read_state     │────►│                                    │
│  - stop_task           │     │  enqueueSdkEvent() ── headless-only│
│  - get_settings        │     │  drainSdkEvents() ── merge到output │
│                        │     │  task_started/progress/notification │
│  output.enqueue(msg)   │     │  session_state_changed             │
│  drainSdkEvents()      │     └────────────────────────────────────┘
└────────────────────────┘
```

---

## 3. 启动流程

### 3.1 从 CLI 入口到 Headless 分支

```
cli.tsx  →  main.tsx  →  print.ts (runHeadless)
```

**第一步：`cli.tsx`** — 通用引导

```typescript
// cli.tsx: 检测 fast paths (--version, daemon, bridge 等)
// 无论 SDK 还是交互模式，都走同一个 bootstrap
await import('../main.js').then(m => m.cliMain())
```

**第二步：`main.tsx`** — 非交互检测

```typescript
const hasPrintFlag = cliArgs.includes('-p') || cliArgs.includes('--print')
const isNonInteractive = hasPrintFlag || hasInitOnlyFlag || hasSdkUrl || !process.stdout.isTTY

// sdkUrl 模式自动启用 stream-json + verbose + print
if (sdkUrl) {
  options.outputFormat = 'stream-json'
  options.verbose = true
  options.print = true
}
```

**第三步：`print.ts`** — Headless 执行

`main.tsx` 的 print 分支动态导入 `src/cli/print.js`，调用 `runHeadless()`。

### 3.2 SDK 初始化序列

```
SDK 消费者                    CLI 进程 (print.ts)
    │                              │
    │  ── stdin: initialize ──►    │  handleInitializeRequest()
    │                              │  - 注册 hooks
    │                              │  - 连接 SDK MCP servers
    │                              │  - 合并 agents
    │                              │  - 配置 systemPrompt
    │  ◄── stdout: init response   │  - 返回 commands/agents/models/account
    │                              │
    │  ── stdin: user message ──►  │  enqueue → run() → ask()
    │  ◄── stdout: system(init)    │  (首轮返回 session 初始化信息)
    │  ◄── stdout: assistant       │
    │  ◄── stdout: tool_progress   │
    │  ◄── stdout: result          │
    │                              │
```

`initialize` 请求 (`SDKControlInitializeRequest`) 包含：

| 字段 | 说明 |
|------|------|
| `hooks` | SDK 回调钩子注册 (按事件类型 → matcher + callback IDs) |
| `sdkMcpServers` | SDK 进程内 MCP 服务器名称列表 |
| `jsonSchema` | 结构化输出的 JSON Schema |
| `systemPrompt` | 自定义系统提示 (避免 ARG_MAX 限制) |
| `appendSystemPrompt` | 追加系统提示 |
| `agents` | SDK 定义的 agent 配置 |
| `promptSuggestions` | 是否启用提示建议 |

初始化响应 (`SDKControlInitializeResponse`) 包含：

| 字段 | 说明 |
|------|------|
| `commands` | 可用的斜杠命令列表 |
| `agents` | 可用的 agent 列表 |
| `models` | 可用模型及其能力 (effort 支持, 自适应思考等) |
| `account` | 账户信息 (邮箱, 组织, 订阅类型, API 提供商) |
| `output_style` | 当前输出风格 |
| `fast_mode_state` | Fast Mode 状态 |

---

## 4. 传输协议

### 4.1 NDJSON (Newline-Delimited JSON)

SDK 模式的核心传输格式是 **NDJSON**——每行一个完整的 JSON 对象，以换行符分隔。

```
{"type":"user","message":{"role":"user","content":"Hello"},"parent_tool_use_id":null}\n
{"type":"assistant","message":{...},"uuid":"...","session_id":"..."}\n
{"type":"result","subtype":"success","result":"Done","duration_ms":1234,...}\n
```

**stdout 保护机制** (`streamJsonStdoutGuard.ts`)：

SDK 消费者按行解析 stdout 为 JSON。任何非 JSON 输出（依赖库的 console.log、调试打印等）会破坏解析器。Guard 机制拦截所有 `process.stdout.write` 调用：
- 合法 JSON 行：正常转发
- 非法行：重定向到 stderr 并标记 `[stdout-guard]`

```typescript
// ndjsonSafeStringify: 确保 JSON 内部不包含裸换行
// 对字符串值中的 \n 进行二次转义
function ndjsonSafeStringify(msg: StdoutMessage): string
```

### 4.2 传输层选择 (Remote 模式)

当使用 `--sdk-url` 连接远程 session 时，`RemoteIO extends StructuredIO` 通过网络传输代替本地 stdin/stdout：

| 环境变量 | 传输方式 | 读 / 写 |
|----------|---------|---------|
| `CLAUDE_CODE_USE_CCR_V2` | SSETransport | SSE reads + POST writes |
| `CLAUDE_CODE_POST_FOR_SESSION_INGRESS_V2` | HybridTransport | WebSocket reads + POST writes |
| (默认) | WebSocketTransport | WebSocket reads + WebSocket writes |

```typescript
// RemoteIO 构造时自动选择传输
const transport = getTransportForUrl(url, headers, sessionId, refreshHeaders)

// 数据到达 → 写入 PassThrough → StructuredIO.read() 消费
transport.setOnData((data: string) => {
  this.inputStream.write(data)
})
```

### 4.3 Direct Connect (WebSocket 回调模式)

`DirectConnectSessionManager` 提供了一种回调式的 SDK 消费方式，用于 VS Code 等需要实时事件的客户端：

```typescript
type DirectConnectCallbacks = {
  onMessage: (message: SDKMessage) => void
  onPermissionRequest: (request, requestId) => void
  onConnected?: () => void
  onDisconnected?: () => void
  onError?: (error: Error) => void
}
```

它通过 WebSocket 连接到远程 session，按行解析 NDJSON，将 `control_request` 路由到权限回调、其余消息路由到通用消息回调。

---

## 5. 消息协议

### 5.1 入站消息 (StdinMessage)

SDK 消费者发送到 CLI 进程的消息类型：

```typescript
StdinMessage = 
  | SDKUserMessage              // 用户对话消息
  | SDKControlRequest           // 控制请求 (初始化、权限响应等)
  | SDKControlResponse          // 控制响应 (对 CLI 发出的请求的回答)
  | SDKKeepAliveMessage         // 保活消息 (维持 WebSocket)
  | SDKUpdateEnvironmentVariables  // 运行时环境变量更新
```

**用户消息** (`SDKUserMessage`)：

```typescript
{
  type: 'user',
  message: { role: 'user', content: '...' },  // Anthropic API 格式
  parent_tool_use_id: null,                     // null = 顶层对话
  uuid?: string,                                // 消息唯一ID
  session_id?: string,
  priority?: 'now' | 'next' | 'later',         // 消息队列优先级
  timestamp?: string                             // ISO 时间戳
}
```

### 5.2 出站消息 (StdoutMessage)

CLI 进程发送给 SDK 消费者的消息类型：

```typescript
StdoutMessage =
  | SDKMessage                              // 所有对话相关消息
  | SDKStreamlinedTextMessage               // 精简文本消息
  | SDKStreamlinedToolUseSummaryMessage     // 精简工具摘要
  | SDKPostTurnSummaryMessage               // 每轮后台摘要
  | SDKControlResponse                      // 控制响应
  | SDKControlRequest                       // 控制请求 (权限等)
  | SDKControlCancelRequest                 // 取消请求
  | SDKKeepAliveMessage                     // 保活
```

**核心 SDKMessage 联合类型** (25+ 变体)：

| 消息类型 | subtype | 说明 |
|----------|---------|------|
| `assistant` | - | 模型回复 (包含完整 API 消息体) |
| `user` | - | 用户消息回放 |
| `result` | `success` | 成功完成，含 cost/usage/duration |
| `result` | `error_*` | 错误终止 (执行错误/超限/超预算) |
| `system` | `init` | Session 初始信息 (工具/模型/MCP等) |
| `system` | `status` | 状态变更 (compacting / 权限模式变更) |
| `system` | `compact_boundary` | Compaction 边界标记 |
| `system` | `task_started` | 后台任务启动 |
| `system` | `task_progress` | 后台任务进度 |
| `system` | `task_notification` | 后台任务终止 |
| `system` | `session_state_changed` | Session 状态变更 (idle/running/requires_action) |
| `system` | `api_retry` | API 重试通知 |
| `system` | `local_command_output` | 本地命令输出 |
| `system` | `hook_started/progress/response` | Hook 生命周期事件 |
| `system` | `elicitation_complete` | MCP elicitation 完成 |
| `system` | `files_persisted` | 文件持久化事件 |
| `stream_event` | - | 流式 token 增量 (需 `includePartialMessages`) |
| `tool_progress` | - | 工具执行进度 |
| `tool_use_summary` | - | 工具使用摘要 |
| `auth_status` | - | 认证状态 (AWS) |
| `rate_limit_event` | - | 速率限制信息 |
| `prompt_suggestion` | - | 下一轮提示建议 |

### 5.3 Result 消息详情

成功 (`SDKResultSuccess`)：

```typescript
{
  type: 'result',
  subtype: 'success',
  result: string,              // 最终文本结果
  duration_ms: number,         // 总耗时
  duration_api_ms: number,     // API 调用耗时
  is_error: boolean,
  num_turns: number,           // 回合数
  total_cost_usd: number,      // 总费用
  usage: NonNullableUsage,     // Token 使用量
  modelUsage: Record<string, ModelUsage>,  // 按模型分列
  permission_denials: SDKPermissionDenial[], // 被拒权限记录
  structured_output?: unknown,  // JSON Schema 结构化输出
  stop_reason: string | null,
}
```

错误 (`SDKResultError`)：

```typescript
{
  type: 'result',
  subtype: 'error_during_execution' | 'error_max_turns' | 'error_max_budget_usd' | 'error_max_structured_output_retries',
  errors: string[],            // 错误信息列表
  // ... 其余字段同 success
}
```

---

## 6. 控制协议 (Control Protocol)

控制协议是 SDK 消费者与 CLI 进程之间的双向 RPC 机制，用于非对话类操作。

### 6.1 请求/响应流程

```
SDK 消费者                              CLI 进程
    │                                      │
    │  (CLI 需要 SDK 消费者做某事)          │
    │  ◄── control_request {               │
    │        request_id: "abc",            │
    │        request: { subtype: "...", }  │
    │      }                               │
    │                                      │
    │  ── control_response {               │
    │       response: {                    │
    │         subtype: "success",          │
    │         request_id: "abc",           │
    │         response: { ... }            │
    │       }                              │
    │     }  ──────────────────────────►   │
    │                                      │
    │  (SDK 消费者向 CLI 发请求)            │
    │  ── control_request {               │
    │       request_id: "xyz",            │
    │       request: { subtype: "...", }  │
    │     }  ──────────────────────────►   │
    │                                      │
    │  ◄── control_response {              │
    │        response: {                   │
    │          subtype: "success",         │
    │          request_id: "xyz",          │
    │          response: { ... }           │
    │        }                             │
    │      }                               │
```

### 6.2 控制请求类型

**CLI → SDK (CLI 发出)：**

| subtype | 说明 | 返回 |
|---------|------|------|
| `can_use_tool` | 请求权限批准 | `PermissionResult` |
| `elicitation` | MCP 用户输入请求 | accept/decline/cancel |
| `hook_callback` | Hook 回调 | `HookJSONOutput` |
| `mcp_message` | MCP JSON-RPC 消息 | JSON-RPC response |

**SDK → CLI (SDK 消费者发出)：**

| subtype | 说明 | 返回 |
|---------|------|------|
| `initialize` | 初始化 session | commands/agents/models/account |
| `interrupt` | 中断当前回合 | (无) |
| `set_permission_mode` | 设置权限模式 | (无) |
| `set_model` | 切换模型 | (无) |
| `set_max_thinking_tokens` | 设置思考 token 上限 | (无) |
| `mcp_status` | 查询 MCP 状态 | `McpServerStatus[]` |
| `mcp_set_servers` | 动态管理 MCP 服务器 | added/removed/errors |
| `mcp_reconnect` | 重连 MCP 服务器 | (无) |
| `mcp_toggle` | 启用/禁用 MCP 服务器 | (无) |
| `get_context_usage` | 查询上下文窗口使用情况 | 详细的 token 分布 |
| `get_settings` | 查询有效配置 | effective + per-source settings |
| `apply_flag_settings` | 应用 flag 设置 | (无) |
| `rewind_files` | 回退文件更改 | canRewind/filesChanged |
| `cancel_async_message` | 取消队列中的消息 | cancelled: boolean |
| `seed_read_state` | 注入文件读取缓存 | (无) |
| `stop_task` | 停止后台任务 | (无) |
| `reload_plugins` | 重载插件 | commands/agents/plugins |

### 6.3 取消机制

```typescript
// CLI 发出取消请求 (当 Hook 先于 SDK 响应做出决定时)
{ type: 'control_cancel_request', request_id: 'abc' }

// SDK 消费者检测到取消时应忽略该 request_id 的后续处理
```

---

## 7. 权限系统 (SDK 模式)

SDK 模式的权限处理是一个精心设计的三层竞速机制：

### 7.1 权限决策流程

```
工具执行请求
     │
     ▼
hasPermissionsToUseTool()  ── 规则引擎评估
     │
     ├── allow → 直接放行
     ├── deny  → 直接拒绝
     │
     └── ask (需要决策)
          │
          ├──────────────────┬────────────────────┐
          ▼                  ▼                    │
   executePermission     sendRequest({           │
   RequestHooks()        subtype:'can_use_tool'  │
   (Hook 评估)           })  (SDK 消费者)         │
          │                  │                    │
          └──► Promise.race ◄┘                    │
                   │                              │
                   ├── Hook 先决定 → abort SDK req │
                   │                              │
                   └── SDK 先响应 → 使用 SDK 结果  │
                                                  │
                   (两者都没有? Hook 无决定          │
                    → 等待 SDK 响应)               │
```

**关键点：** 权限不是无条件批准。即使在 SDK 模式下：

1. **规则引擎优先**：声明式 allow/deny 规则先执行
2. **Hook 与 SDK 竞速**：PermissionRequest Hook 和 SDK 权限回调并行运行
3. **先到先得**：哪个先返回就使用哪个的决定，另一个被取消
4. **SDK URL 强制 stdio**：使用 `--sdk-url` 时，权限总是委托给 SDK 消费者

### 7.2 SDK 权限回调

```typescript
// CLI → SDK 的权限请求
{
  type: 'control_request',
  request_id: '...',
  request: {
    subtype: 'can_use_tool',
    tool_name: 'Write',
    input: { file_path: '/src/main.ts', content: '...' },
    permission_suggestions: [...],  // 建议的权限规则
    blocked_path: '/etc/passwd',    // 被阻止的路径
    decision_reason: '...',         // 决策原因
    tool_use_id: '...',
    agent_id: '...',                // 如果在子 agent 中
    description: 'Write to /src/main.ts'
  }
}

// SDK → CLI 的权限响应
{
  type: 'control_response',
  response: {
    subtype: 'success',
    request_id: '...',
    response: {
      behavior: 'allow',         // allow | deny
      updatedInput: { ... },     // 可选修改输入
      updatedPermissions: [...], // 可选持久化规则
      toolUseID: '...',
      decisionClassification: 'user_temporary'  // 遥测分类
    }
  }
}
```

### 7.3 权限拒绝追踪

`QueryEngine` 包装 `canUseTool` 以记录所有拒绝，最终体现在 `result` 消息的 `permission_denials` 数组中：

```typescript
const wrappedCanUseTool: CanUseToolFn = async (...) => {
  const result = await canUseTool(...)
  if (result.behavior !== 'allow') {
    this.permissionDenials.push({
      tool_name: sdkCompatToolName(tool.name),
      tool_use_id: toolUseID,
      tool_input: input,
    })
  }
  return result
}
```

---

## 8. SDK 事件队列

`sdkEventQueue.ts` 是 Headless 模式专用的内部事件中转站，用于将异步系统事件注入到 SDK 输出流中。

### 8.1 事件类型

```typescript
type SdkEvent =
  | TaskStartedEvent           // 后台任务启动
  | TaskProgressEvent          // 后台任务进度 (含 token 使用和工具摘要)
  | TaskNotificationSdkEvent   // 后台任务终止 (completed/failed/stopped)
  | SessionStateChangedEvent   // Session 状态变更 (idle/running/requires_action)
```

### 8.2 工作机制

```typescript
// 仅在 Headless 模式下入队 (交互模式忽略)
export function enqueueSdkEvent(event: SdkEvent): void {
  if (!getIsNonInteractiveSession()) return
  if (queue.length >= MAX_QUEUE_SIZE) queue.shift()  // 上限 1000
  queue.push(event)
}

// print.ts 在每轮查询完成后排空队列，合并到 output stream
export function drainSdkEvents(): Array<SdkEvent & { uuid, session_id }> {
  return queue.splice(0).map(e => ({
    ...e, uuid: randomUUID(), session_id: getSessionId()
  }))
}
```

**`session_state_changed('idle')` 是权威的"回合结束"信号**——它在 `result` 消息之后、所有后台 agent 清理完毕后才发出。SDK 消费者应依赖此信号而非 `result` 来判断 session 真正空闲。

---

## 9. MCP 桥接 (SDK MCP Transport)

SDK 模式支持在 SDK 进程内运行 MCP 服务器，通过控制协议与 CLI 进程中的 MCP 客户端桥接。

### 9.1 架构

```
CLI 进程                                SDK 进程
┌─────────────────────┐                ┌──────────────────────┐
│  MCP Client         │                │  MCP Server          │
│       │             │                │       ▲              │
│       ▼             │                │       │              │
│  SdkControlClient   │ control_req    │  SdkControlServer    │
│  Transport          │ ──────────►    │  Transport           │
│       │             │ stdout→stdin   │       │              │
│       │             │                │       │              │
│       │             │ control_resp   │       │              │
│       │             │ ◄────────────  │       │              │
│       ▼             │ stdin→stdout   │       ▼              │
│  (result)           │                │  (response)          │
└─────────────────────┘                └──────────────────────┘
```

### 9.2 消息流

1. CLI 的 MCP Client 调用工具 → JSONRPC 请求发送到 `SdkControlClientTransport`
2. Transport 包装为 `control_request { subtype: 'mcp_message', server_name, message }`
3. 通过 stdout 发送到 SDK 进程
4. SDK 的 `SdkControlServerTransport` 接收，转发给 MCP Server
5. MCP Server 处理后返回结果
6. 结果通过 `control_response` 返回给 CLI

### 9.3 SDK MCP Server 创建

```typescript
// agentSdkTypes.ts — 公开 API (存根，实际实现在构建时注入)
export function createSdkMcpServer(options: {
  name: string
  version?: string
  tools?: Array<SdkMcpToolDefinition<any>>
}): McpSdkServerConfigWithInstance

// tool() 辅助函数定义 MCP 工具
export function tool<Schema>(
  name: string,
  description: string,
  inputSchema: Schema,    // Zod schema
  handler: (args, extra) => Promise<CallToolResult>,
  extras?: { annotations?, searchHint?, alwaysLoad? }
): SdkMcpToolDefinition<Schema>
```

---

## 10. 输出格式

### 10.1 三种输出模式

| `--output-format` | 行为 |
|-------------------|------|
| `text` (默认) | 只输出最终 `result.result` 文本 |
| `json` | 输出最终 `result` 的 JSON (或 `--verbose` 时输出所有消息的 JSON 数组) |
| `stream-json` | 实时 NDJSON 流，每个消息一行 |

### 10.2 Streamlined 模式

当 `CLAUDE_CODE_STREAMLINED_OUTPUT=true` 且 `--output-format=stream-json` 时，消息会被转换为精简格式：

```typescript
// 完整 assistant 消息 → 精简文本
{ type: 'streamlined_text', text: '...', session_id, uuid }

// 工具使用 → 精简摘要
{ type: 'streamlined_tool_use_summary', tool_summary: 'Read 2 files, wrote 1 file', ... }
```

这减少了带宽并简化了 SDK 消费者的处理逻辑。

### 10.3 流式 Token 事件

当 `--include-partial-messages` 启用时，`QueryEngine` 会 yield `stream_event` 消息，包含原始 Anthropic streaming event。这允许 SDK 消费者实现逐字符的流式显示。

---

## 11. Session 管理 API

`agentSdkTypes.ts` 定义了 Session 管理的公开 API（存根实现，实际实现在构建产物中）：

### 11.1 核心 API

```typescript
// 单次查询 (v1 API)
function query(params: {
  prompt: string | AsyncIterable<SDKUserMessage>
  options?: Options
}): Query  // AsyncIterable<SDKMessage>

// 创建持久 session (v2 API, unstable)
function unstable_v2_createSession(options: SDKSessionOptions): SDKSession

// 恢复已有 session
function unstable_v2_resumeSession(sessionId: string, options): SDKSession

// 单次便捷查询
async function unstable_v2_prompt(message: string, options): Promise<SDKResultMessage>
```

### 11.2 Session 操作

```typescript
// 列出 sessions
async function listSessions(options?: { dir?, limit?, offset? }): Promise<SDKSessionInfo[]>

// 获取 session 信息
async function getSessionInfo(sessionId: string, options?): Promise<SDKSessionInfo | undefined>

// 读取 session 消息历史
async function getSessionMessages(sessionId: string, options?): Promise<SessionMessage[]>

// 重命名 session
async function renameSession(sessionId: string, title: string, options?): Promise<void>

// 标记 session
async function tagSession(sessionId: string, tag: string | null, options?): Promise<void>

// 分叉 session
async function forkSession(sessionId: string, options?): Promise<{ sessionId: string }>
```

### 11.3 Daemon 原语 (Internal)

```typescript
// 守护进程桥接 (claude.ai 远程控制)
async function connectRemoteControl(opts: ConnectRemoteControlOptions): Promise<RemoteControlHandle | null>

// 定时任务监控
function watchScheduledTasks(opts: {
  dir: string, signal: AbortSignal, getJitterConfig?
}): ScheduledTasksHandle
```

---

## 12. QueryEngine 中的 SDK 适配

QueryEngine 有多个 SDK 专用配置选项：

```typescript
type QueryEngineConfig = {
  // SDK 专用选项
  setSDKStatus?: (status: SDKStatus) => void     // 状态回调 (如 'compacting')
  includePartialMessages?: boolean                // 是否 yield stream_event
  replayUserMessages?: boolean                    // 消息回放
  snipReplay?: (yielded, store) => ...            // 长 session 内存优化
  orphanedPermission?: PermissionDecision         // 恢复中断的权限决策
  
  // 通用选项 (SDK 也使用)
  maxTurns?: number
  maxBudgetUsd?: number
  taskBudget?: { total: number }
  jsonSchema?: Record<string, unknown>            // 结构化输出
}
```

**关键差异：**

| 方面 | 交互模式 | SDK 模式 |
|------|---------|---------|
| UI | React/Ink TUI | 无 UI |
| 权限 | 终端弹窗 | control_request + SDK 回调 |
| 输出 | Ink 渲染 | NDJSON stdout |
| Trust | 显式信任对话 | 隐式信任 |
| Hooks | 根据 trust 可能跳过 | 总是执行 |
| processUserInput | querySource: 'repl' | querySource: 'sdk' |
| 调试输出 | 允许 | 禁止 (debug: false，保护 stdout) |
| 消息历史 | 完整保留 (UI 需要) | snipReplay 可截断 (节省内存) |

---

## 13. Hooks 系统集成

SDK 模式完整支持 Hooks 系统，且有专门的回调机制：

### 13.1 SDK Hook 回调

SDK 消费者可以通过 `initialize` 请求注册 Hook 回调：

```typescript
{
  subtype: 'initialize',
  hooks: {
    'PreToolUse': [{
      matcher: 'Write',           // 工具名匹配
      hookCallbackIds: ['cb_1'],  // 回调 ID
      timeout: 30000              // 超时
    }],
    'PostToolUse': [{ ... }]
  }
}
```

当 Hook 事件触发时，CLI 通过 `control_request` 发送回调：

```typescript
{
  type: 'control_request',
  request_id: '...',
  request: {
    subtype: 'hook_callback',
    callback_id: 'cb_1',
    input: { hook_event_name: 'PreToolUse', tool_name: 'Write', ... },
    tool_use_id: '...'
  }
}
```

SDK 消费者返回 `HookJSONOutput` 决定行为（允许/拒绝/修改输入等）。

### 13.2 Hook 事件列表 (27 种)

完整的 Hook 事件包括工具生命周期 (`PreToolUse`, `PostToolUse`, `PostToolUseFailure`)、权限 (`PermissionRequest`, `PermissionDenied`)、Session 生命周期 (`SessionStart`, `SessionEnd`, `Stop`, `StopFailure`)、Sub-agent (`SubagentStart`, `SubagentStop`)、Compaction (`PreCompact`, `PostCompact`)、通知 (`Notification`)、配置变更 (`ConfigChange`, `InstructionsLoaded`)、文件系统 (`FileChanged`, `CwdChanged`, `WorktreeCreate`, `WorktreeRemove`)、Team (`TeammateIdle`, `TaskCreated`, `TaskCompleted`)、MCP (`Elicitation`, `ElicitationResult`)、启动 (`Setup`, `UserPromptSubmit`) 等。

---

## 14. 类型体系

### 14.1 类型生成架构

```
coreSchemas.ts (Zod schemas, SSOT)
         │
         ▼  bun scripts/generate-sdk-types.ts
coreTypes.generated.ts (TypeScript types)
         │
         ▼  re-export
coreTypes.ts → agentSdkTypes.ts → SDK 消费者
```

SDK 的类型体系采用 **Zod Schema 作为单一真相源 (SSOT)**：

- `coreSchemas.ts`：定义所有可序列化的数据类型 (消息、配置、权限等)
- `controlSchemas.ts`：定义控制协议类型 (请求/响应包装)
- 通过 `generate-sdk-types.ts` 脚本生成 TypeScript 类型
- `coreTypes.ts` 和 `agentSdkTypes.ts` 作为公开导出入口

### 14.2 关键类型文件

| 文件 | 角色 | 受众 |
|------|------|------|
| `coreSchemas.ts` | Zod SSOT，运行时校验 | 内部 |
| `coreTypes.ts` | 生成的 TS 类型 + 常量 | SDK 消费者 |
| `controlSchemas.ts` | 控制协议 Zod 定义 | SDK 构建者 |
| `agentSdkTypes.ts` | 公开 API 入口 + 函数存根 | SDK 消费者 |
| `sandboxTypes.ts` | 沙箱配置类型 | 两者 |

### 14.3 缺失的生成文件

此代码库中以下文件被引用但不存在（它们是构建时生成的）：

- `coreTypes.generated.ts` — 从 Zod 生成的核心类型
- `controlTypes.ts` — 从 controlSchemas 生成的控制类型
- `runtimeTypes.ts` — 非可序列化类型（回调、接口）
- `sdkUtilityTypes.ts` — 工具类型
- `settingsTypes.generated.ts` — 设置类型
- `toolTypes.ts` — 工具类型

---

## 15. StructuredIO 详解

`StructuredIO` 是 SDK 通信的核心类，封装了所有 stdin/stdout 交互逻辑。

### 15.1 核心组件

```typescript
class StructuredIO {
  readonly structuredInput: AsyncGenerator<StdinMessage | SDKMessage>
  readonly outbound: Stream<StdoutMessage>  // 单一 FIFO 输出通道
  
  private pendingRequests: Map<string, PendingRequest<unknown>>
  private resolvedToolUseIds: Set<string>  // 防重复响应，上限 1000
  
  // 公开方法
  write(message: StdoutMessage): Promise<void>
  createCanUseTool(onPermissionPrompt?): CanUseToolFn
  handleElicitation(serverName, params, signal): Promise<ElicitResult>
  sendMcpMessage(serverName, message): Promise<JSONRPCMessage>
  injectControlResponse(response): void  // Bridge 注入权限响应
  prependUserMessage(content): void      // 预置用户消息
  getPendingPermissionRequests(): SDKControlRequest[]
}
```

### 15.2 输入解析 (`read()`)

```typescript
private async *read() {
  // NDJSON 逐行解析
  for await (const block of this.input) {
    content += block
    // 按 \n 分割，每行 JSON.parse
    // 特殊处理：
    //   keep_alive → 静默忽略
    //   update_environment_variables → 直接应用到 process.env
    //   control_response → 匹配 pendingRequests 并 resolve
    //   user/control_request → yield 给 print.ts 主循环
  }
}
```

### 15.3 输出合并

所有输出通过单一 `outbound` Stream 合并：
- `sendRequest()` 的 `control_request` 
- `print.ts` 的 `output.enqueue()` (assistant, result, system events)
- `drainSdkEvents()` 的系统事件

这保证了消息的 FIFO 顺序，防止控制请求超越排队中的流事件。

---

## 16. 错误处理与恢复

### 16.1 断开重连

`RemoteIO` 通过 Transport 层的自动重连处理网络中断：
- WebSocket 断开后自动重连
- `refreshHeaders` 回调在重连时获取新 token
- `keep_alive` 消息维持连接活跃

### 16.2 中断恢复

```typescript
// 自动恢复中断的回合
if (turnInterruptionState && resumeInterruptedTurnEnv) {
  removeInterruptedMessage(mutableMessages, turnInterruptionState.message)
  enqueue({
    mode: 'prompt',
    value: turnInterruptionState.message.message.content,
    uuid: randomUUID()
  })
}
```

### 16.3 优雅关闭

`SIGINT` → abort 当前查询 → `gracefulShutdown(0)` → 持久化 session 状态 → flush analytics → 超时强制退出。

### 16.4 Orphaned Permission

当 session 恢复时，可能存在"孤儿"权限请求（CLI 已发出 `can_use_tool` 但未收到响应就中断了）。`orphanedPermission` 参数允许 SDK 消费者在恢复时提供之前的决策。

---

## 17. 关键设计决策

### 17.1 单入口复用

**决策：** 不创建独立的 SDK 入口文件，复用 CLI 的 `cli.tsx → main.tsx → print.ts` 路径。

**原因：** 避免两套代码路径的功能漂移。所有工具、权限检查、compaction、MCP 集成等核心逻辑在 SDK 和交互模式间完全共享。

### 17.2 NDJSON 而非 WebSocket/gRPC

**决策：** 本地 SDK 使用 stdin/stdout NDJSON，远程使用 WebSocket/SSE 上的同一 NDJSON 协议。

**原因：** NDJSON 是最简单的流式 JSON 格式，没有帧头，任何语言都能解析。stdin/stdout 是进程间通信的最普遍方式。

### 17.3 Zod Schema 作为类型 SSOT

**决策：** 用 Zod 定义数据模型，生成 TypeScript 类型，而非反过来。

**原因：** Zod schema 同时提供运行时校验和静态类型，消除了手写类型与实际数据不一致的风险。

### 17.4 Hook 与 SDK 权限竞速

**决策：** PermissionRequest Hook 和 SDK `can_use_tool` 回调并行运行，先到先得。

**原因：** 避免 Hook 阻塞 UI 响应（或反过来）。这与交互模式中 Hook 与终端提示并行的行为一致。

### 17.5 单一输出 FIFO

**决策：** 所有 stdout 消息通过单一 `Stream<StdoutMessage>` 排队。

**原因：** 防止控制请求和对话消息交错导致的协议违规。保证消费者按序处理。

---

## 18. 环境变量参考

| 变量 | 说明 |
|------|------|
| `CLAUDE_CODE_USE_CCR_V2` | 启用 SSE + POST 传输 |
| `CLAUDE_CODE_POST_FOR_SESSION_INGRESS_V2` | 启用 Hybrid 传输 |
| `CLAUDE_CODE_STREAMLINED_OUTPUT` | 启用精简输出转换 |
| `CLAUDE_CODE_RESUME_INTERRUPTED_TURN` | 自动恢复中断回合 |
| `CLAUDE_CODE_INCLUDE_PARTIAL_MESSAGES` | 启用 stream_event |
| `CLAUDE_CODE_STREAM_CLOSE_TIMEOUT` | 流关闭超时 (默认 60s) |
| `CLAUDE_CODE_PROFILE_STARTUP` | Headless 启动性能分析 |
| `CLAUDE_CODE_ENVIRONMENT_KIND` | 环境类型 (bridge/remote) |
| `CLAUDE_CODE_SESSION_ACCESS_TOKEN` | Session 访问令牌 |
