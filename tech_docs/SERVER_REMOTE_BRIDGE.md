---
title: Server / Remote / Bridge
aliases: [远程模式, Server Remote Bridge]
series: Claude Code 架构解析
category: 运行模式
order: 17
tags:
  - claude-code
  - server
  - remote
  - bridge
  - websocket
date: 2026-04-04
---

# Server / Remote / Bridge 详细技术文档

## 1. 概述

Claude Code 支持三种远程/服务端运行模式，每种针对不同的使用场景：

| 模式 | 命令 | 说明 |
|------|------|------|
| **Direct Connect Server** | `claude server` | 本地 HTTP/WebSocket 服务器，IDE 和客户端通过 `cc://` URL 连接 |
| **Remote Control (Bridge)** | `claude remote-control` | 注册本地环境为 CCR Worker，claude.ai/code 远程控制 |
| **CCR Remote Session** | `claude --remote` / `--teleport` | 本地 TUI 连接到云端 CCR 容器执行 |

### 核心依赖关系

```
┌───────────────────────────────────────────────────────────────────────┐
│                        三种远程模式                                     │
│                                                                         │
│  ┌─────────────────┐  ┌──────────────────┐  ┌─────────────────────┐  │
│  │ Direct Connect  │  │ Remote Control   │  │ CCR Remote Session │  │
│  │ (claude server) │  │ (bridge)         │  │ (--remote)         │  │
│  │                 │  │                  │  │                    │  │
│  │ 自托管 HTTP/WS  │  │ 注册为 Worker    │  │ 连接到云端容器     │  │
│  │ IDE/客户端连接  │  │ claude.ai 控制   │  │ 本地 TUI 展示     │  │
│  └────────┬────────┘  └────────┬─────────┘  └────────┬────────────┘  │
│           │                    │                     │                │
│           ▼                    ▼                     ▼                │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │               共享基础设施                                        │ │
│  │                                                                    │ │
│  │  SDK 消息协议 (NDJSON)    Control Request/Response                │ │
│  │  权限代理                  Session 生命周期管理                    │ │
│  │  WebSocket 传输            sdkMessageAdapter 消息转换             │ │
│  └──────────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────────┘
```

---

## 2. Direct Connect Server

### 2.1 架构

```
┌──────────────┐     HTTP POST /sessions      ┌──────────────────────┐
│  IDE / 客户端 │ ──────────────────────────► │  claude server       │
│  (cc://...)  │                               │                      │
│              │ ◄────────────────────────── │  { session_id,       │
│              │  { session_id, ws_url }       │    ws_url }          │
│              │                               │                      │
│              │     WebSocket (NDJSON)         │  SessionManager      │
│              │ ◄──────────────────────────► │    → ChildProcess    │
│              │  SDK messages + control       │    → session 隔离    │
└──────────────┘                               └──────────────────────┘
```

### 2.2 服务端配置

```typescript
// src/server/types.ts
type ServerConfig = {
  port: number               // 监听端口
  host: string               // 绑定地址
  authToken: string          // Bearer 认证令牌
  unix?: string              // Unix Domain Socket 路径
  idleTimeoutMs?: number     // 空闲会话超时 (0 = 永不过期)
  maxSessions?: number       // 最大并发会话数
  workspace?: string         // 默认工作目录
}
```

启动命令：

```bash
claude server --port 3000 --host 127.0.0.1 --auth-token <token> \
  --max-sessions 5 --idle-timeout 3600000 --workspace /path/to/code
```

### 2.3 会话生命周期

```typescript
type SessionState = 'starting' | 'running' | 'detached' | 'stopping' | 'stopped'

type SessionInfo = {
  id: string
  status: SessionState
  createdAt: number
  workDir: string
  process: ChildProcess | null
  sessionKey?: string
}
```

**会话持久化**：`~/.claude/server-sessions.json` 存储 `SessionIndex`，支持跨服务器重启恢复：

```typescript
type SessionIndexEntry = {
  sessionId: string
  transcriptSessionId: string  // 用于 --resume
  cwd: string
  permissionMode?: string
  createdAt: number
  lastActiveAt: number
}
```

### 2.4 连接协议

**Step 1: 创建会话 (HTTP)**

```typescript
// createDirectConnectSession.ts
POST ${serverUrl}/sessions
Headers: { Authorization: "Bearer <token>", Content-Type: "application/json" }
Body: { cwd: "/path/to/project", dangerously_skip_permissions?: true }

Response (200): {
  session_id: "sess_abc123",
  ws_url: "ws://localhost:3000/sessions/sess_abc123/ws",
  work_dir: "/path/to/project"
}
```

**Step 2: WebSocket 连接**

```typescript
// directConnectManager.ts
class DirectConnectSessionManager {
  connect(): void {
    this.ws = new WebSocket(this.config.wsUrl, {
      headers: { authorization: `Bearer ${this.config.authToken}` }
    })
  }
}
```

**Step 3: 消息交换 (NDJSON)**

```
← 服务端 → 客户端:
  SDK messages (assistant, result, system, tool_use, ...)
  control_request (can_use_tool 权限请求)
  keep_alive

→ 客户端 → 服务端:
  user messages (type: 'user')
  control_response (allow/deny)
  control_request (interrupt)
```

### 2.5 URL 协议

```
cc://hostname:port?token=xxx          → HTTP
cc+unix:///path/to/socket?token=xxx   → Unix Domain Socket
```

`main.tsx` 中 `cc://` URL 被早期解析为 `serverUrl` + `authToken`，然后决定是进入交互式 TUI 还是 headless 模式。

---

## 3. Remote Control (Bridge)

### 3.1 架构

```
┌─────────────────┐                    ┌──────────────────────┐
│  claude.ai/code │ ← WebSocket/SSE → │  CCR Backend         │
│  (浏览器)       │                    │  (Cloud)             │
└─────────────────┘                    └──────────┬───────────┘
                                                  │ Poll / Event
                                                  ▼
                                       ┌──────────────────────┐
                                       │  claude remote-control│
                                       │  (Bridge Daemon)      │
                                       │                       │
                                       │  runBridgeLoop()      │
                                       │  ├─ poll backend      │
                                       │  ├─ dequeue work      │
                                       │  └─ spawn sessions    │
                                       └──────────┬───────────┘
                                                  │ spawn
                                       ┌──────────▼───────────┐
                                       │  Child Claude CLI     │
                                       │  --sdk-url <ingress>  │
                                       │  --print (headless)   │
                                       │  RemoteIO transport   │
                                       └──────────────────────┘
```

### 3.2 启用门控

```typescript
// bridgeEnabled.ts
export function isBridgeEnabled(): boolean {
  return feature('BRIDGE_MODE')
    ? isClaudeAISubscriber() &&
        getFeatureValue_CACHED_MAY_BE_STALE('tengu_ccr_bridge', false)
    : false
}
```

**门控层级**：

1. **编译时**: `feature('BRIDGE_MODE')` — DCE 排除
2. **订阅检查**: `isClaudeAISubscriber()` — 需要 claude.ai 账户
3. **GrowthBook**: `tengu_ccr_bridge` — 渐进式发布
4. **OAuth scope**: 需要 `user:profile` scope（完整登录而非 setup-token）
5. **组织信息**: 需要 `organizationUuid`（用于 GrowthBook 定向）
6. **策略**: `allow_remote_sessions` 企业策略

### 3.3 Bridge 主循环

```typescript
// bridgeMain.ts
export async function bridgeMain(args: string[]): Promise<void> {
  // 1. OAuth 认证
  // 2. 创建 BridgeApiClient
  // 3. 注册环境 (registerWorker)
  // 4. runBridgeLoop()
}
```

**`runBridgeLoop`** 是核心事件循环：

- 轮询 CCR 后端获取待处理的 work
- 使用 `SessionSpawner` 创建子进程执行 work
- 管理会话生命周期（启动、运行、超时、停止）
- 处理多会话（`tengu_ccr_bridge_multi_session` 门控）

**退避策略**：

```typescript
const DEFAULT_BACKOFF = {
  connInitialMs: 2_000,       // 连接错误初始延迟
  connCapMs: 120_000,         // 连接错误上限 (2 分钟)
  connGiveUpMs: 600_000,      // 连接错误放弃 (10 分钟)
  generalInitialMs: 500,      // 通用错误初始延迟
  generalCapMs: 30_000,       // 通用错误上限
  generalGiveUpMs: 600_000,   // 通用错误放弃
}
```

### 3.4 会话派生 (Session Spawner)

```typescript
// sessionRunner.ts
function createSessionSpawner(deps: SessionSpawnerDeps): SessionSpawner {
  return {
    spawn(opts: SessionSpawnOpts): SessionHandle { ... }
  }
}
```

子进程参数：

```bash
claude --print --sdk-url <ingress-url> --session-id <id> \
  --input-format stream-json --output-format stream-json \
  --replay-user-messages
```

**关键环境变量**：

| 变量 | 说明 |
|------|------|
| `CLAUDE_CODE_ENVIRONMENT_KIND=bridge` | 标识 bridge 环境 |
| `CLAUDE_CODE_SESSION_ACCESS_TOKEN` | 会话级别 OAuth token |
| `CLAUDE_CODE_POST_FOR_SESSION_INGRESS_V2=1` | v1: Hybrid 传输 |
| `CLAUDE_CODE_USE_CCR_V2=1` | v2: SSE 传输 |
| `CLAUDE_CODE_WORKER_EPOCH` | Worker 纪元 (v2) |
| `CLAUDE_CODE_FORCE_SANDBOX=1` | 强制沙箱 |

**子进程通信**：Bridge 通过 stdin/stdout 的 NDJSON 与子进程通信：

```typescript
// 子进程 stdout → Bridge 解析
const rl = createInterface({ input: child.stdout })
rl.on('line', line => {
  const parsed = jsonParse(line)
  if (parsed.type === 'control_request') {
    // 转发权限请求到 CCR
    deps.onPermissionRequest?.(sessionId, parsed, accessToken)
  }
})
```

### 3.5 传输层

三种传输可选，由环境变量决定：

```typescript
// transportUtils.ts
function getTransportForUrl(url, headers, sessionId, refreshHeaders): Transport {
  if (CLAUDE_CODE_USE_CCR_V2) {
    return new SSETransport(...)      // v2: SSE 读 + HTTP POST 写
  }
  if (CLAUDE_CODE_POST_FOR_SESSION_INGRESS_V2) {
    return new HybridTransport(...)   // v1+: WS 读 + HTTP POST 写
  }
  return new WebSocketTransport(...)  // 默认: WS 读 + WS 写
}
```

| 传输 | 读 | 写 | 场景 |
|------|----|----|------|
| **WebSocketTransport** | WebSocket | WebSocket | 默认 |
| **HybridTransport** | WebSocket | HTTP POST | v1 桥接 (fire-and-forget 写入) |
| **SSETransport** | SSE (EventSource) | HTTP POST | v2 CCR |

### 3.6 RemoteIO

```typescript
// src/cli/remoteIO.ts
class RemoteIO extends StructuredIO {
  // 连接到 --sdk-url 的传输层
  // 桥接模式: 周期性 keep_alive + echo control_request 到 stdout
  // CCR v2: 接线 CCRClient + 内部事件 + 生命周期
}
```

`RemoteIO` 是 `StructuredIO` 的子类，专门用于远程 transport（SSE/WS/Hybrid），添加了桥接特有逻辑。

---

## 4. CCR Remote Session

### 4.1 架构

```
┌─────────────────┐                     ┌──────────────────────┐
│  本地 CLI (TUI)  │ ← WebSocket ───── │  CCR API             │
│  --remote        │  /v1/sessions/ws/  │  (Anthropic Cloud)   │
│  --teleport      │  {id}/subscribe    │                      │
│                  │                     │  容器/Worker 中运行  │
│  RemoteSession   │ ── HTTP POST ────► │  Claude agent        │
│  Manager         │  /v1/sessions/     │                      │
│                  │  {id}/events        │                      │
└─────────────────┘                     └──────────────────────┘
```

### 4.2 SessionsWebSocket

```typescript
// src/remote/SessionsWebSocket.ts
class SessionsWebSocket {
  // 连接: wss://<BASE_API_URL>/v1/sessions/ws/{sessionId}/subscribe?organization_uuid=...
  // 认证: Authorization: Bearer <accessToken> (Header)
  // 协议: SDKMessage + control_request/response

  // 重连策略:
  //   MAX_RECONNECT_ATTEMPTS = 5
  //   RECONNECT_DELAY_MS = 2000
  //   PING_INTERVAL_MS = 30000
  //   永久关闭: 4003 (unauthorized)
  //   临时关闭: 4001 (session not found, 最多重试 3 次，compaction 期间)
}
```

**WebSocket Close Codes**：

| Code | 含义 | 行为 |
|------|------|------|
| `4003` | Unauthorized | 永久关闭，停止重连 |
| `4001` | Session not found | 有限重试 (最多 3 次，compaction 期间可能暂时丢失) |
| 其他 | 临时断开 | 标准重连 (最多 5 次，2s 延迟) |

### 4.3 RemoteSessionManager

```typescript
// src/remote/RemoteSessionManager.ts
class RemoteSessionManager {
  // 协调:
  //   - WebSocket 订阅: 接收 CCR 消息
  //   - HTTP POST: 发送用户消息
  //   - 权限请求/响应流程

  config: RemoteSessionConfig = {
    sessionId: string
    getAccessToken: () => string
    orgUuid: string
    hasInitialPrompt?: boolean
    viewerOnly?: boolean  // claude assistant 附加模式
  }
}
```

**消息发送**：

```typescript
// 用户消息通过 HTTP POST 发送
sendEventToRemoteSession(sessionId, content, orgUuid, accessToken)
// → POST /v1/sessions/{id}/events
// Headers: Authorization, anthropic-version, beta: ccr-byoc-2025-07-29
```

**viewerOnly 模式** (`claude assistant` 附加)：
- 不发送 interrupt
- 无 60s 权限响应超时
- 不更新 session title

### 4.4 权限代理

```typescript
type RemotePermissionResponse =
  | { behavior: 'allow', updatedInput: Record<string, unknown> }
  | { behavior: 'deny', message: string }
```

远程会话的权限请求通过 `control_request (can_use_tool)` 从 CCR 发送到本地 TUI，用户在本地审批后通过 `control_response` 回复。

---

## 5. 消息适配器

### 5.1 sdkMessageAdapter

```typescript
// src/remote/sdkMessageAdapter.ts
export function convertSDKMessage(
  message: SDKMessage,
  options: { convertToolResults?: boolean, convertUserTextMessages?: boolean }
): ConvertedMessage | 'ignored'
```

将 CCR WebSocket 收到的 `SDKMessage`（云端格式）转换为本地 REPL 可以渲染的 `Message` / `StreamEvent`。被 `useRemoteSession` 和 `useDirectConnect` 共用。

### 5.2 remotePermissionBridge

```typescript
// src/remote/remotePermissionBridge.ts
export function createSyntheticAssistantMessage(...)
export function createToolStub(...)
```

当工具在远端执行时，构建合成的 `AssistantMessage` + `Tool` 对象用于本地权限 UI 展示。

---

## 6. Bridge 组件详解

### 6.1 文件索引 (src/bridge/)

| 文件 | 角色 |
|------|------|
| `bridgeMain.ts` | 主入口: OAuth → 注册 → runBridgeLoop |
| `bridgeEnabled.ts` | 门控: 编译/GrowthBook/订阅/OAuth/策略 |
| `bridgeApi.ts` | HTTP 客户端: CCR/bridge 后端 API |
| `sessionRunner.ts` | 子进程 spawn + stdin/stdout NDJSON 通信 |
| `bridgeMessaging.ts` | 入站消息解析 + control 协议 |
| `bridgePermissionCallbacks.ts` | 权限流程 |
| `replBridge.ts` | REPL ↔ Bridge 核心接线 |
| `replBridgeTransport.ts` | v1 Hybrid vs v2 SSE 传输适配 |
| `initReplBridge.ts` | REPL bridge 初始化入口 |
| `workSecret.ts` | CCR v2 SDK URL / Worker 注册 |
| `createSession.ts` | 创建/更新 bridge 会话 |
| `sessionIdCompat.ts` | `cse_*` ↔ `session_*` ID 转换 |
| `jwtUtils.ts` | JWT token 刷新调度 |
| `trustedDevice.ts` | 可信设备 token |
| `pollConfig.ts` | GrowthBook 轮询/keepalive 间隔 |
| `capacityWake.ts` | 容量唤醒 |
| `bridgeUI.ts` | 交互式 bridge UI / 日志 |
| `types.ts` | 类型: SessionHandle, BridgeConfig, SpawnMode |

### 6.2 Bridge API 交互

```
claude remote-control
        │
        ▼
1. OAuth 认证 (claude.ai 账户)
        │
        ▼
2. registerWorker() → 注册为 CCR Worker
   POST /v1/code/workers
   { hostname, cwd, capabilities }
        │
        ▼
3. runBridgeLoop() → 轮询循环
   ┌─ GET /v1/code/workers/{id}/work → 获取待处理 work
   │
   ├─ work available?
   │  YES → createSessionSpawner().spawn(opts)
   │         → child CLI process (--sdk-url, --print)
   │
   ├─ 权限请求?
   │  child stdout → control_request → 转发到 CCR
   │  CCR response → control_response → child stdin
   │
   └─ session 结束?
       → stopWorkWithRetry() → 通知 CCR
```

### 6.3 多会话支持

```typescript
const SPAWN_SESSIONS_DEFAULT = 32  // 默认最大派生数

async function isMultiSessionSpawnEnabled(): Promise<boolean> {
  return checkGate_CACHED_OR_BLOCKING('tengu_ccr_bridge_multi_session')
}
```

多会话模式 (`--spawn` / `--capacity` / `--create-session-in-dir`) 由 `tengu_ccr_bridge_multi_session` 门控，允许单个 bridge daemon 同时管理多个 Claude 会话。

---

## 7. 认证体系

### 7.1 Direct Connect

```
Client → Server: Authorization: Bearer <authToken>
  - HTTP 创建会话时
  - WebSocket 连接时 (upgrade headers)
  - authToken 来自 cc:// URL 的 query parameter 或 --auth-token
```

### 7.2 Bridge (Remote Control)

```
Bridge → CCR: OAuth token (claude.ai 账户)
  - 注册 Worker
  - 轮询 work
  - JWT 刷新

子进程: CLAUDE_CODE_SESSION_ACCESS_TOKEN
  - 每会话独立的 access token
  - 子进程的 OAuth token 被清除，防止越权
```

### 7.3 CCR Remote Session

```
Local TUI → CCR:
  WebSocket: Authorization: Bearer <oauthAccessToken>
             + anthropic-version header
             + organization_uuid query parameter

  HTTP POST: prepareApiRequest() headers
             + beta header: ccr-byoc-2025-07-29
```

---

## 8. 进程模型

### 8.1 Direct Connect

```
claude server (主进程)
├── SessionManager
│   ├── Session 1: ChildProcess (claude --print --sdk-url ...)
│   ├── Session 2: ChildProcess
│   └── Session N: ChildProcess (受 maxSessions 限制)
└── HTTP/WS Server
    ├── POST /sessions → 创建会话
    └── WS /sessions/{id}/ws → 消息路由
```

### 8.2 Bridge

```
claude remote-control (daemon 进程)
├── BridgeApiClient (CCR 通信)
├── TokenRefreshScheduler (JWT 刷新)
├── SessionSpawner
│   ├── Session 1: ChildProcess (--sdk-url <ingress>)
│   │   ├── stdin: 用户消息 / control_response
│   │   └── stdout: SDK messages / control_request
│   ├── Session 2: ChildProcess
│   └── Session N: ChildProcess (受 SPAWN_SESSIONS_DEFAULT 限制)
└── runBridgeLoop (轮询 + 调度)
```

### 8.3 CCR Remote

```
claude --remote (本地 TUI)
├── RemoteSessionManager
│   ├── SessionsWebSocket (接收)
│   └── HTTP POST (发送)
├── Permission Queue (权限审批)
└── REPL (渲染)

╌╌╌╌╌╌ 网络边界 ╌╌╌╌╌╌

CCR Container (云端)
├── Claude Agent (完整工具链)
└── SDK streaming (NDJSON)
```

---

## 9. 消息过滤

Direct Connect 和 CCR 都过滤部分消息类型：

```typescript
// directConnectManager.ts — 过滤后才回调 onMessage
const FILTERED_TYPES = [
  'control_response',          // 自己发的响应
  'keep_alive',                // 心跳
  'control_cancel_request',    // 取消
  'streamlined_text',          // 精简文本 (重复)
  'streamlined_tool_use_summary', // 精简工具摘要
  'system' + 'post_turn_summary', // 系统 + turn 总结
]
```

---

## 10. UDS Messaging

独立于 WebSocket/HTTP 的进程间通信通道：

```bash
claude --messaging-socket-path /tmp/claude.sock
```

基于 Unix Domain Socket 的收件箱，由 `UDS_INBOX` feature flag 门控。用于跨进程直接通信，与 CCR WebSocket/SSE 路径正交。

---

## 11. 客户端类型检测

```typescript
// main.tsx — 基于环境判断客户端类型
function getClientType(): string {
  if (CLAUDE_CODE_ENTRYPOINT === 'sdk-ts')         return 'sdk-typescript'
  if (CLAUDE_CODE_ENTRYPOINT === 'sdk-py')          return 'sdk-python'
  if (CLAUDE_CODE_ENTRYPOINT === 'sdk-cli')         return 'sdk-cli'
  if (CLAUDE_CODE_ENTRYPOINT === 'claude-vscode')   return 'claude-vscode'
  if (CLAUDE_CODE_ENTRYPOINT === 'local-agent')     return 'local-agent'
  if (CLAUDE_CODE_ENTRYPOINT === 'claude-desktop')  return 'claude-desktop'
  if (CLAUDE_CODE_ENTRYPOINT === 'remote' || hasSessionIngressToken)
                                                     return 'remote'
  return 'cli'
}
```

Bridge 子进程被标记为 `CLAUDE_CODE_ENVIRONMENT_KIND = 'bridge'`，作为 session source `'remote-control'`。

---

## 12. 关键文件索引

### Server (src/server/)

| 文件 | 角色 |
|------|------|
| `types.ts` | ServerConfig, SessionState, SessionInfo, SessionIndex |
| `createDirectConnectSession.ts` | POST /sessions 创建会话 |
| `directConnectManager.ts` | DirectConnectSessionManager (WS 客户端) |
| `server.ts` (未在仓库中) | HTTP/WS 服务器实现 |
| `sessionManager.ts` (未在仓库中) | 会话管理器 |

### Remote (src/remote/)

| 文件 | 角色 |
|------|------|
| `RemoteSessionManager.ts` | CCR 远程会话管理 (WS + HTTP) |
| `SessionsWebSocket.ts` | CCR WebSocket 客户端 (/v1/sessions/ws/) |
| `sdkMessageAdapter.ts` | SDKMessage → REPL Message 转换 |
| `remotePermissionBridge.ts` | 合成权限 UI 对象 |

### Bridge (src/bridge/)

| 文件 | 角色 |
|------|------|
| `bridgeMain.ts` | Bridge daemon 主入口 + runBridgeLoop |
| `bridgeEnabled.ts` | 多层门控 (编译/GrowthBook/订阅/策略) |
| `sessionRunner.ts` | 子进程 spawn + NDJSON 通信 |
| `bridgeApi.ts` | CCR HTTP 客户端 |
| `bridgeMessaging.ts` | 消息解析 + control 协议 |
| `workSecret.ts` | CCR v2 SDK URL / Worker 注册 |
| `types.ts` | SessionHandle, BridgeConfig, SpawnMode |

### 传输 (src/cli/transports/)

| 文件 | 角色 |
|------|------|
| `transportUtils.ts` | 传输选择 (SSE/Hybrid/WebSocket) |
| `SSETransport.ts` | SSE 读 + HTTP POST 写 (v2) |
| `HybridTransport.ts` | WS 读 + HTTP POST 写 (v1+) |
| `WebSocketTransport.ts` | WS 读 + WS 写 (默认) |

### 共享

| 文件 | 角色 |
|------|------|
| `src/cli/remoteIO.ts` | RemoteIO (StructuredIO 子类) |
| `src/cli/print.ts` | headless runner |

---

## 13. 关键设计决策

### 13.1 NDJSON 统一协议

**决策：** 所有三种远程模式共享同一套 SDK NDJSON 消息格式和 control request/response 协议。

**原因：** 统一的消息格式使得 `sdkMessageAdapter`、权限桥接、消息过滤等基础设施可以被所有模式复用。新增远程模式只需实现传输层。

### 13.2 子进程隔离

**决策：** Bridge 和 Direct Connect Server 都通过 `spawn()` 创建独立子进程执行每个会话。

**原因：** 
- 进程隔离确保会话间不共享状态
- 崩溃不影响其他会话
- 可以独立设置环境变量（如 session token）
- 子进程的 OAuth token 被清除，防止越权

### 13.3 三种传输策略

**决策：** SSE (v2) / Hybrid (v1+) / WebSocket (默认)，由环境变量选择。

**原因：**
- **SSE**: 单向服务端推送 + HTTP POST 写入，适合 CCR v2 的无状态后端
- **Hybrid**: WS 读取 + HTTP POST 写入，解决桥接 fire-and-forget 写入的可靠性
- **WebSocket**: 全双工低延迟，适合默认直连场景

### 13.4 会话状态持久化

**决策：** Direct Connect Server 在 `~/.claude/server-sessions.json` 持久化会话索引。

**原因：** 支持服务器重启后恢复活跃会话。`transcriptSessionId` 允许使用 `--resume` 恢复对话上下文。

### 13.5 viewerOnly 模式

**决策：** `RemoteSessionManager` 支持 `viewerOnly` 模式（`claude assistant` 附加）。

**原因：** 观察者不应发送 interrupt（会中断正在执行的任务）、不应触发权限超时（无人审批）、不应更新 session title（由创建者控制）。
