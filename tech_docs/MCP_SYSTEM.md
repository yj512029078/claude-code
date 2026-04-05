---
title: Claude Code MCP 系统
aliases: [MCP 系统, MCP System, Model Context Protocol]
series: Claude Code 架构解析
category: 扩展生态
order: 12
tags:
  - claude-code
  - mcp
  - protocol
  - oauth
  - tool-integration
date: 2026-04-04
---

# Claude Code MCP 系统 (Model Context Protocol) — 深度技术文档

## 目录

1. [架构概览](#1-架构概览)
2. [配置加载系统](#2-配置加载系统)
   - 2.1 [多层配置来源与优先级](#21-多层配置来源与优先级)
   - 2.2 [传输协议类型](#22-传输协议类型)
   - 2.3 [配置文件格式与校验](#23-配置文件格式与校验)
   - 2.4 [策略过滤 (Allowlist / Denylist)](#24-策略过滤-allowlist--denylist)
   - 2.5 [去重机制](#25-去重机制)
3. [连接建立与生命周期](#3-连接建立与生命周期)
   - 3.1 [Transport 创建](#31-transport-创建)
   - 3.2 [Client 初始化与握手](#32-client-初始化与握手)
   - 3.3 [连接状态机](#33-连接状态机)
   - 3.4 [连接缓存与重连](#34-连接缓存与重连)
   - 3.5 [错误处理与断线重连](#35-错误处理与断线重连)
4. [工具发现机制](#4-工具发现机制)
   - 4.1 [内置工具静态注册](#41-内置工具静态注册)
   - 4.2 [MCP 工具动态发现](#42-mcp-工具动态发现)
   - 4.3 [MCP 工具到内部 Tool 格式的转换](#43-mcp-工具到内部-tool-格式的转换)
   - 4.4 [工具池组装与合并](#44-工具池组装与合并)
5. [工具延迟加载 (Tool Search)](#5-工具延迟加载-tool-search)
   - 5.1 [延迟判定规则](#51-延迟判定规则)
   - 5.2 [自动启用阈值](#52-自动启用阈值)
   - 5.3 [ToolSearchTool 实现](#53-toolsearchtool-实现)
   - 5.4 [tool_reference 机制](#54-tool_reference-机制)
   - 5.5 [Deferred Tools Delta 通知](#55-deferred-tools-delta-通知)
6. [工具调用流程](#6-工具调用流程)
   - 6.1 [调用链路](#61-调用链路)
   - 6.2 [callMCPTool 核心实现](#62-callmcptool-核心实现)
   - 6.3 [Elicitation 处理](#63-elicitation-处理)
   - 6.4 [超时与取消](#64-超时与取消)
7. [结果处理](#7-结果处理)
   - 7.1 [结果转换](#71-结果转换)
   - 7.2 [大结果处理](#72-大结果处理)
   - 7.3 [图片结果处理](#73-图片结果处理)
8. [工具 Schema 发送到 LLM](#8-工具-schema-发送到-llm)
   - 8.1 [toolToAPISchema 转换](#81-tooltoapischema-转换)
   - 8.2 [defer_loading 标记](#82-defer_loading-标记)
   - 8.3 [Schema 缓存](#83-schema-缓存)
9. [资源与命令发现](#9-资源与命令发现)
10. [认证与 OAuth](#10-认证与-oauth)
11. [连接管理 UI (React)](#11-连接管理-ui-react)
12. [关键数据流总结](#12-关键数据流总结)

---

## 1. 架构概览

MCP (Model Context Protocol) 是 Claude Code 的**外部工具扩展协议**。Claude Code 作为 MCP **Client**，通过标准 JSON-RPC 2.0 与外部 MCP Server 通信，动态发现和调用第三方工具。

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Claude Code (MCP Client)                     │
│                                                                      │
│  ┌────────────┐  ┌──────────────┐  ┌─────────────┐  ┌────────────┐ │
│  │ config.ts  │  │  client.ts   │  │  MCPTool.ts  │  │ toolSearch │ │
│  │ 配置加载    │→│ 连接管理/缓存 │→│ 工具转换/调用 │→│ 延迟加载    │ │
│  └────────────┘  └──────────────┘  └─────────────┘  └────────────┘ │
│         │              │                  │                │         │
│         │              │     JSON-RPC 2.0 │                │         │
│         │       ┌──────┴──────┐           │                │         │
│         │       │  Transport  │           │                │         │
│         │       │ stdio/SSE/  │           │                │         │
│         │       │ HTTP/WS/SDK │           │                │         │
│         │       └──────┬──────┘           │                │         │
└─────────│──────────────│──────────────────│────────────────│─────────┘
          │              │                  │                │
          ▼              ▼                  ▼                ▼
   .mcp.json        MCP Servers       tools/list         LLM API
   settings.json    (外部进程/服务)    tools/call      (defer_loading)
```

核心源码文件：

| 文件 | 职责 |
|------|------|
| `src/services/mcp/types.ts` | 类型定义 — 配置 Schema、连接状态、序列化接口 |
| `src/services/mcp/config.ts` | 配置加载 — 多 scope 合并、策略过滤、去重 |
| `src/services/mcp/client.ts` | 核心实现 — 连接、工具发现、工具调用（~3300 行） |
| `src/tools/MCPTool/MCPTool.ts` | MCP 工具模板 — 被 client.ts 克隆覆盖 |
| `src/tools/ToolSearchTool/ToolSearchTool.ts` | 延迟工具搜索 |
| `src/utils/toolSearch.ts` | Tool Search 策略判定与阈值计算 |
| `src/services/mcp/MCPConnectionManager.tsx` | React Context — 连接管理 UI |
| `src/services/mcp/useManageMCPConnections.ts` | React Hook — 连接生命周期 |
| `src/tools.ts` | 工具池组装 — 内置 + MCP 合并 |
| `src/utils/api.ts` | Schema 转换 — Tool → API 参数 |

---

## 2. 配置加载系统

### 2.1 多层配置来源与优先级

配置加载在 `src/services/mcp/config.ts` 中实现，合并优先级从低到高：

```
plugin → user → project → local → enterprise
```

| 来源 (scope) | 文件位置 | 说明 |
|-------------|---------|------|
| `plugin` | 插件内置 | 通过 `getPluginMcpServers()` 加载，key 前缀为 `plugin:name:server` |
| `user` | `~/.claude/settings.json` → `mcpServers` | 全局用户配置 |
| `project` | `CWD/.mcp.json`（向上递归查找） | 项目级配置，需要审批 (`approved`) |
| `local` | `.claude/settings.local.json` → `mcpServers` | 当前项目本地配置 |
| `enterprise` | `managed-mcp.json` | 企业管控，**拥有排他性控制权** |
| `dynamic` | 运行时传入（如 `--mcp-config`） | 动态配置 |
| `claudeai` | claude.ai API 获取 | claude.ai 代理连接器 |

**核心加载函数**：

```typescript
// 快速路径 — 仅本地文件读取，无网络调用
async function getClaudeCodeMcpConfigs(
  dynamicServers, extraDedupTargets
): Promise<{ servers, errors }>

// 完整路径 — 含 claude.ai 网络获取
async function getAllMcpConfigs(): Promise<{ servers, errors }>
```

`getClaudeCodeMcpConfigs()` 是启动时的快速路径，只做本地文件读取；`getAllMcpConfigs()` 额外发起 claude.ai 连接器的网络获取，两者并行以减少延迟。

**企业排他模式**：当 `managed-mcp.json` 存在时，`doesEnterpriseMcpConfigExist()` 返回 true，此时**只加载企业配置**，忽略所有用户/项目/插件配置。

### 2.2 传输协议类型

```typescript
// src/services/mcp/types.ts
export const TransportSchema = z.enum([
  'stdio',      // 子进程 — stdin/stdout
  'sse',        // Server-Sent Events
  'sse-ide',    // IDE 专用 SSE（内部）
  'http',       // Streamable HTTP (新标准)
  'ws',         // WebSocket
  'sdk',        // 进程内 SDK 传输
])
```

各传输类型的配置 Schema：

```typescript
// stdio — 最常见，启动子进程
McpStdioServerConfig = {
  type?: 'stdio',          // 可省略（向后兼容）
  command: string,         // 如 "npx"
  args: string[],          // 如 ["-y", "@slack/mcp-server"]
  env?: Record<string, string>,
}

// http — Streamable HTTP (推荐的远程协议)
McpHTTPServerConfig = {
  type: 'http',
  url: string,
  headers?: Record<string, string>,
  headersHelper?: string,  // 动态 header 脚本
  oauth?: { clientId?, callbackPort?, authServerMetadataUrl?, xaa? },
}

// sse — SSE 长连接
McpSSEServerConfig = {
  type: 'sse',
  url: string,
  headers?: Record<string, string>,
  oauth?: { ... },
}

// ws — WebSocket
McpWebSocketServerConfig = {
  type: 'ws',
  url: string,
  headers?: Record<string, string>,
}

// sdk — 进程内（IDE 扩展用）
McpSdkServerConfig = {
  type: 'sdk',
  name: string,
}
```

### 2.3 配置文件格式与校验

`.mcp.json` 文件格式：

```json
{
  "mcpServers": {
    "slack": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@anthropic/slack-mcp"],
      "env": { "SLACK_TOKEN": "${SLACK_TOKEN}" }
    },
    "github": {
      "type": "http",
      "url": "https://mcp.github.com/sse"
    }
  }
}
```

校验通过 Zod Schema `McpJsonConfigSchema` 完成。环境变量通过 `expandEnvVars()` 展开，支持 `${VAR}` 语法，缺失变量产生 warning 级别错误。

### 2.4 策略过滤 (Allowlist / Denylist)

企业策略可以控制哪些 MCP 服务器允许/禁止连接：

```typescript
// Denylist 优先 — 绝对禁止
function isMcpServerDenied(serverName, config?): boolean
  // 检查 name / command / URL 匹配

// Allowlist — 允许列表（空列表 = 全部禁止）
function isMcpServerAllowedByPolicy(serverName, config?): boolean
  // 1. 先检查 denylist（绝对优先）
  // 2. 无 allowlist → 允许所有
  // 3. 有 allowlist → 必须匹配 name/command/URL
```

`allowedMcpServers` 支持三种匹配模式：
- **name 匹配**：`{ serverName: "slack" }`
- **command 匹配**：`{ serverCommand: ["npx", "-y", "@slack/mcp"] }`
- **URL 匹配**（支持通配符）：`{ serverUrl: "https://*.example.com/*" }`

### 2.5 去重机制

两种去重场景：

**Plugin vs Manual 去重** (`dedupPluginMcpServers`)：
- 插件服务器 key 为 `plugin:name:server`，与手动配置不会 key 冲突
- 通过**签名比较**（command 数组或 URL）检测实际相同的服务器
- 手动配置优先；插件间 first-loaded 优先

**Claude.ai 连接器去重** (`dedupClaudeAiMcpServers`)：
- 连接器 key 为 `claude.ai DisplayName`，与手动配置不冲突
- 仅对已启用的手动服务器做签名去重
- CCR 代理 URL 会解包 `mcp_url` 查询参数还原原始 URL

签名计算：
```typescript
function getMcpServerSignature(config): string | null {
  const cmd = getServerCommandArray(config)
  if (cmd) return `stdio:${JSON.stringify(cmd)}`
  const url = getServerUrl(config)
  if (url) return `url:${unwrapCcrProxyUrl(url)}`
  return null  // sdk 类型无签名
}
```

---

## 3. 连接建立与生命周期

### 3.1 Transport 创建

`connectToServer`（`src/services/mcp/client.ts:595`）是连接入口，使用 `memoize` 缓存连接实例：

```typescript
export const connectToServer = memoize(
  async (name, serverRef, serverStats?): Promise<MCPServerConnection> => {
    let transport

    switch (serverRef.type) {
      case 'stdio':
        transport = new StdioClientTransport({
          command: serverRef.command,
          args: serverRef.args,
          env: { ...subprocessEnv(), ...serverRef.env },
          stderr: 'pipe',
        })

      case 'sse':
        const authProvider = new ClaudeAuthProvider(name, serverRef)
        transport = new SSEClientTransport(new URL(serverRef.url), {
          authProvider,
          fetch: wrapFetchWithTimeout(wrapFetchWithStepUpDetection(...)),
          requestInit: { headers: { 'User-Agent': ... } },
          eventSourceInit: { /* 长连接不设超时 */ },
        })

      case 'http':
        transport = new StreamableHTTPClientTransport(new URL(serverRef.url), {
          authProvider,
          fetch: wrapFetchWithTimeout(...),
          requestInit: { headers: { ... } },
        })

      case 'ws':
        const wsClient = new WebSocket(serverRef.url, { protocols: ['mcp'] })
        transport = new WebSocketTransport(wsClient)

      case 'sdk':
        // 由 setupSdkMcpClients 处理，不走 connectToServer

      // 特殊情况：Chrome MCP / Computer Use → 进程内运行
      case 'stdio' && isClaudeInChromeMCPServer(name):
        const [clientTransport, serverTransport] = createLinkedTransportPair()
        inProcessServer = createClaudeForChromeMcpServer(context)
        await inProcessServer.connect(serverTransport)
        transport = clientTransport
    }
    // ...
  },
  getServerCacheKey,  // key = `${name}-${JSON.stringify(serverRef)}`
)
```

### 3.2 Client 初始化与握手

Transport 创建后，使用 MCP SDK 的 `Client` 完成协议握手：

```typescript
// 1. 创建 Client 实例
const client = new Client(
  {
    name: 'claude-code',
    title: 'Claude Code',
    version: MACRO.VERSION ?? 'unknown',
  },
  {
    capabilities: {
      roots: {},         // 支持 ListRoots
      elicitation: {},   // 支持 Elicitation
    },
  },
)

// 2. 注册 ListRoots handler — 告知服务器工作目录
client.setRequestHandler(ListRootsRequestSchema, async () => ({
  roots: [{ uri: `file://${getOriginalCwd()}` }],
}))

// 3. 注册默认 Elicitation handler（初始化阶段返回 cancel）
client.setRequestHandler(ElicitRequestSchema, async () => ({
  action: 'cancel',
}))

// 4. 连接（带超时保护）
await Promise.race([
  client.connect(transport),
  new Promise((_, reject) => setTimeout(() => reject(...), connectionTimeoutMs)),
])

// 5. 读取服务器能力和信息
const capabilities = client.getServerCapabilities()
const serverVersion = client.getServerVersion()
const instructions = client.getInstructions()
```

### 3.3 连接状态机

```typescript
// src/services/mcp/types.ts
type MCPServerConnection =
  | ConnectedMCPServer   // 已连接，client 可用
  | FailedMCPServer      // 连接失败，error 描述原因
  | NeedsAuthMCPServer   // 需要 OAuth 认证
  | PendingMCPServer     // 正在连接/重连中
  | DisabledMCPServer    // 用户禁用
```

状态转换：

```
                    ┌──── Disabled (用户手动禁用)
                    │
  Config Loaded ────┼──── Pending ──────┬──── Connected
                    │       ↑           │       │
                    │       │           │    onerror/onclose
                    │       │           │       │
                    │    reconnect      ▼       ▼
                    │       │        Failed   Pending (重连)
                    │       └──────────┘         │
                    │                     超过 MAX_RECONNECT_ATTEMPTS
                    │                            │
                    └──── NeedsAuth ◄────────────┘
                          (401/OAuth)     (或 401 错误)
```

### 3.4 连接缓存与重连

连接通过 `memoize` 缓存，key 为 `${name}-${JSON.stringify(config)}`：

```typescript
// 确保有效连接（tool 调用前）
export async function ensureConnectedClient(client): Promise<ConnectedMCPServer> {
  if (client.config.type === 'sdk') return client  // SDK 走进程内
  const connected = await connectToServer(client.name, client.config)
  if (connected.type !== 'connected') throw Error('not connected')
  return connected
}

// 清除缓存 → 下次调用自动重连
export async function clearServerCache(name, config): Promise<void> {
  // 关闭现有连接
  const existing = await connectToServer(name, config)
  if (existing.type === 'connected') await existing.cleanup()
  // 清除所有缓存
  connectToServer.cache.delete(key)
  fetchToolsForClient.cache.delete(name)
  fetchResourcesForClient.cache.delete(name)
  fetchCommandsForClient.cache.delete(name)
}
```

### 3.5 错误处理与断线重连

**连接断开检测**：

```typescript
// 连续终端错误计数 → 触发 close
let consecutiveConnectionErrors = 0
const MAX_ERRORS_BEFORE_RECONNECT = 3

client.onerror = (error) => {
  if (isTerminalConnectionError(error.message)) {
    consecutiveConnectionErrors++
    if (consecutiveConnectionErrors >= MAX_ERRORS_BEFORE_RECONNECT) {
      closeTransportAndRejectPending('too many connection errors')
    }
  }
}

// onclose 触发缓存清除 → 下次工具调用自动重连
client.onclose = () => {
  connectToServer.cache.delete(cacheKey)
  fetchToolsForClient.cache.delete(name)
  // ...
}
```

**终端错误识别**：
- `ECONNRESET` / `ETIMEDOUT` / `EPIPE` / `EHOSTUNREACH` / `ECONNREFUSED`
- `Body Timeout Error` / `terminated`
- `SSE stream disconnected` / `Failed to reconnect SSE stream`

**重连策略**（`useManageMCPConnections.ts`）：
- 指数退避：初始 1s，最大 30s
- 最多重连 5 次 (`MAX_RECONNECT_ATTEMPTS`)
- 超时后标记为 `failed`

---

## 4. 工具发现机制

### 4.1 内置工具静态注册

`src/tools.ts` 中 `getAllBaseTools()` 是所有内置工具的**注册入口**：

```typescript
export function getAllBaseTools(): Tools {
  return [
    // 核心工具 — 始终可用
    AgentTool, TaskOutputTool, BashTool,
    GlobTool, GrepTool,         // 搜索（无嵌入工具时）
    FileReadTool, FileEditTool, FileWriteTool,
    NotebookEditTool, WebFetchTool, WebSearchTool,
    TodoWriteTool, TaskStopTool, AskUserQuestionTool,
    SkillTool, EnterPlanModeTool, ExitPlanModeV2Tool,
    BriefTool, SendMessageTool,

    // 条件工具 — feature flag / 环境变量控制
    ...(isTodoV2Enabled() ? [TaskCreateTool, TaskGetTool, ...] : []),
    ...(isWorktreeModeEnabled() ? [EnterWorktreeTool, ExitWorktreeTool] : []),
    ...(isToolSearchEnabledOptimistic() ? [ToolSearchTool] : []),
    // ...更多条件工具
  ]
}
```

`getTools()` 在 `getAllBaseTools()` 基础上应用过滤：

```typescript
export const getTools = (permissionContext): Tools => {
  // 1. Simple 模式 → 仅 Bash + Read + Edit
  if (process.env.CLAUDE_CODE_SIMPLE) { /* minimal set */ }

  // 2. 排除特殊工具（由 MCP 连接时动态添加）
  const tools = getAllBaseTools().filter(t => !specialTools.has(t.name))

  // 3. Deny 规则过滤
  let allowed = filterToolsByDenyRules(tools, permissionContext)

  // 4. REPL 模式过滤（隐藏原始工具）
  if (isReplModeEnabled()) { /* filter REPL_ONLY_TOOLS */ }

  // 5. isEnabled() 检查
  return allowed.filter(t => t.isEnabled())
}
```

### 4.2 MCP 工具动态发现

MCP 工具通过 JSON-RPC 的 `tools/list` 方法从服务器获取：

```typescript
// src/services/mcp/client.ts — fetchToolsForClient
export const fetchToolsForClient = memoizeWithLRU(
  async (client: MCPServerConnection): Promise<Tool[]> => {
    if (client.type !== 'connected') return []
    if (!client.capabilities?.tools) return []

    // 发送 JSON-RPC: { method: 'tools/list' }
    const result = await client.client.request(
      { method: 'tools/list' },
      ListToolsResultSchema,
    )

    // 清理 Unicode，转换格式
    const toolsToProcess = recursivelySanitizeUnicode(result.tools)
    return toolsToProcess.map(tool => /* 转换为内部 Tool 格式 */)
  },
  (client) => client.name,
  MCP_FETCH_CACHE_SIZE,  // LRU cache size = 20
)
```

**批量连接流程** (`getMcpToolsCommandsAndResources`)：

```typescript
export async function getMcpToolsCommandsAndResources(
  onConnectionAttempt, mcpConfigs?
): Promise<void> {
  const allConfigEntries = Object.entries(mcpConfigs ?? (await getAllMcpConfigs()).servers)

  // 1. 分离 disabled 服务器（不建立连接）
  for (const entry of allConfigEntries) {
    if (isMcpServerDisabled(entry[0])) {
      onConnectionAttempt({ client: { type: 'disabled' }, tools: [], commands: [] })
    } else {
      configEntries.push(entry)
    }
  }

  // 2. 按类型分组，不同并发度
  const localServers = configEntries.filter(([_, c]) => isLocalMcpServer(c))   // stdio/sdk
  const remoteServers = configEntries.filter(([_, c]) => !isLocalMcpServer(c)) // sse/http/ws

  // 3. 并行处理每个服务器
  const processServer = async ([name, config]) => {
    // 跳过 needs-auth 缓存的服务器
    if (await isMcpAuthCached(name)) { /* 返回 needs-auth + McpAuthTool */ }

    // 连接
    const client = await connectToServer(name, config, serverStats)

    // 获取工具、命令、技能、资源
    const [tools, commands, skills, resources] = await Promise.all([
      fetchToolsForClient(client),
      fetchCommandsForClient(client),
      fetchMcpSkillsForClient(client),       // skill:// 资源
      fetchResourcesForClient(client),        // 标准资源
    ])

    onConnectionAttempt({ client, tools: [...tools, ...resourceTools], commands, resources })
  }

  // 4. 本地和远程分别并发
  await Promise.all([
    processBatched(localServers, localBatchSize, processServer),
    processBatched(remoteServers, remoteBatchSize, processServer),
  ])
}
```

### 4.3 MCP 工具到内部 Tool 格式的转换

每个 MCP 服务器工具都从 `MCPTool` 模板克隆并覆盖关键字段：

```typescript
return {
  ...MCPTool,                                    // 基础模板
  name: buildMcpToolName(client.name, tool.name), // "mcp__slack__search"
  mcpInfo: { serverName: client.name, toolName: tool.name },
  isMcp: true,

  // 从 MCP 服务器获取的元数据
  searchHint: tool._meta?.['anthropic/searchHint'],   // 搜索提示词
  alwaysLoad: tool._meta?.['anthropic/alwaysLoad'],   // 是否立即加载

  // 描述和 Schema 直接透传
  async description() { return tool.description ?? '' },
  inputJSONSchema: tool.inputSchema,

  // Annotations 映射
  isConcurrencySafe() { return tool.annotations?.readOnlyHint ?? false },
  isReadOnly()        { return tool.annotations?.readOnlyHint ?? false },
  isDestructive()     { return tool.annotations?.destructiveHint ?? false },
  isOpenWorld()       { return tool.annotations?.openWorldHint ?? false },

  // 权限 — passthrough（需要用户确认）
  async checkPermissions() {
    return { behavior: 'passthrough', message: 'MCPTool requires permission.' }
  },

  // 调用 — 实际执行 MCP tool call
  async call(args, context, _, parentMessage, onProgress) {
    const connectedClient = await ensureConnectedClient(client)
    return await callMCPToolWithUrlElicitationRetry({ ... })
  },

  // 显示名称
  userFacingName() {
    const displayName = tool.annotations?.title || tool.name
    return `${client.name} - ${displayName} (MCP)`
  },
}
```

### 4.4 工具池组装与合并

`assembleToolPool()`（`src/tools.ts:345`）是最终组装函数：

```typescript
export function assembleToolPool(permissionContext, mcpTools): Tools {
  const builtInTools = getTools(permissionContext)
  const allowedMcpTools = filterToolsByDenyRules(mcpTools, permissionContext)

  // 排序规则：内置在前 + MCP 在后，各自按名称字母序
  // uniqBy 去重：内置优先（同名时保留内置）
  const byName = (a, b) => a.name.localeCompare(b.name)
  return uniqBy(
    [...builtInTools].sort(byName).concat(allowedMcpTools.sort(byName)),
    'name',
  )
}
```

REPL 中由 `useMergedTools` hook 使用，headless 由 `mergeAndFilterTools` 使用：

```typescript
// src/utils/toolPool.ts
export function mergeAndFilterTools(initialTools, assembled, mode): Tools {
  // 1. initialTools 优先去重
  const merged = uniqBy([...initialTools, ...assembled], 'name')

  // 2. 分区排序：内置 prefix + MCP suffix
  const [mcp, builtIn] = partition(merged, isMcpTool)
  const tools = [...builtIn.sort(byName), ...mcp.sort(byName)]

  // 3. Coordinator 模式过滤
  if (isCoordinatorMode()) {
    return applyCoordinatorToolFilter(tools)
  }
  return tools
}
```

---

## 5. 工具延迟加载 (Tool Search)

### 5.1 延迟判定规则

```typescript
// src/tools/ToolSearchTool/prompt.ts
export function isDeferredTool(tool: Tool): boolean {
  // 1. alwaysLoad 显式免除（MCP _meta['anthropic/alwaysLoad']）
  if (tool.alwaysLoad === true) return false

  // 2. MCP 工具 → 始终延迟
  if (tool.isMcp) return true

  // 3. 内置工具 → 由 shouldDefer 标记决定
  return tool.shouldDefer === true
}
```

### 5.2 自动启用阈值

Tool Search 有三种模式：

| 模式 | `ENABLE_TOOL_SEARCH` | 行为 |
|------|---------------------|------|
| `tst` | `true` / `auto:0` / 未设置 | **始终延迟** MCP 和 shouldDefer 工具 |
| `tst-auto` | `auto` / `auto:N` (1-99) | 超过阈值时延迟 |
| `standard` | `false` / `auto:100` | 不延迟，所有工具直接加载 |

自动阈值计算：
```typescript
// 默认：上下文窗口的 10%
const DEFAULT_AUTO_TOOL_SEARCH_PERCENTAGE = 10

function getAutoToolSearchTokenThreshold(model): number {
  const contextWindow = getContextWindowForModel(model, betas)
  const percentage = getAutoToolSearchPercentage() / 100
  return Math.floor(contextWindow * percentage)
}
```

还有前置检查：
- 模型必须支持 `tool_reference`（Haiku 不支持）
- `ToolSearchTool` 必须在工具列表中（未被 deny 规则禁用）
- 非第一方 API 代理默认禁用（除非显式设置 `ENABLE_TOOL_SEARCH`）

### 5.3 ToolSearchTool 实现

```typescript
// src/tools/ToolSearchTool/ToolSearchTool.ts
export const ToolSearchTool = buildTool({
  name: 'ToolSearch',

  async call(input, { options: { tools }, getAppState }) {
    const { query, max_results = 5 } = input
    const deferredTools = tools.filter(isDeferredTool)

    // 方式一：直接选择 — "select:mcp__slack__search"
    //   支持多选："select:A,B,C"
    const selectMatch = query.match(/^select:(.+)$/i)
    if (selectMatch) {
      return findByExactName(selectMatch[1], deferredTools)
    }

    // 方式二：关键词搜索
    //   - 精确匹配：工具名 / mcp__ 前缀
    //   - 评分搜索：name parts + description + searchHint
    //   - 支持 +required 语法
    const matches = await searchToolsWithKeywords(query, deferredTools, tools, max_results)
    return buildSearchResult(matches, query, deferredTools.length)
  },
})
```

**搜索评分规则**：

| 匹配类型 | 分数 |
|---------|------|
| MCP 工具 name part 精确匹配 | 12 |
| 非 MCP 工具 name part 精确匹配 | 10 |
| MCP 工具 name part 包含匹配 | 6 |
| 非 MCP 工具 name part 包含匹配 | 5 |
| searchHint 词边界匹配 | 4 |
| full name 包含匹配 | 3 |
| description 词边界匹配 | 2 |

### 5.4 tool_reference 机制

ToolSearchTool 的结果通过 `tool_reference` content block 返回：

```typescript
mapToolResultToToolResultBlockParam(content, toolUseID) {
  return {
    type: 'tool_result',
    tool_use_id: toolUseID,
    content: content.matches.map(name => ({
      type: 'tool_reference',
      tool_name: name,
    })),
  }
}
```

API 收到 `tool_reference` 后，会自动将对应工具的完整 schema 注入到模型上下文中，使模型可以在后续 turn 中调用这些工具。

**跨 turn 追踪** — `extractDiscoveredToolNames()`：

```typescript
// 扫描消息历史中的 tool_reference，重建已发现工具集
export function extractDiscoveredToolNames(messages): Set<string> {
  const discovered = new Set<string>()
  for (const msg of messages) {
    // compact boundary 携带的历史发现集
    if (msg.type === 'system' && msg.subtype === 'compact_boundary') {
      for (const name of msg.compactMetadata?.preCompactDiscoveredTools ?? [])
        discovered.add(name)
    }
    // 从 tool_result 中提取 tool_reference
    if (msg.type === 'user') {
      for (const block of msg.message?.content ?? []) {
        if (isToolResultBlockWithContent(block)) {
          for (const item of block.content) {
            if (isToolReferenceWithName(item)) discovered.add(item.tool_name)
          }
        }
      }
    }
  }
  return discovered
}
```

### 5.5 Deferred Tools Delta 通知

新增/移除的延迟工具通过 `deferred_tools_delta` attachment 通知模型：

```typescript
export function getDeferredToolsDelta(tools, messages): DeferredToolsDelta | null {
  // 1. 扫描已宣告的工具（从历史 attachment 中）
  const announced = new Set<string>()
  for (const msg of messages) {
    if (msg.attachment?.type === 'deferred_tools_delta') {
      for (const n of msg.attachment.addedNames) announced.add(n)
      for (const n of msg.attachment.removedNames) announced.delete(n)
    }
  }

  // 2. 计算差异
  const added = deferred.filter(t => !announced.has(t.name))
  const removed = [...announced].filter(n => !poolNames.has(n))

  if (added.length === 0 && removed.length === 0) return null
  return { addedNames, addedLines, removedNames }
}
```

---

## 6. 工具调用流程

### 6.1 调用链路

```
LLM 返回 tool_use block
        │
        ▼
Tool.call(args, context)              ← src/services/mcp/client.ts:1833
        │
        ├── extractToolUseId(parentMessage)
        ├── onProgress({ status: 'started' })
        │
        ▼
ensureConnectedClient(client)          ← 确保连接有效
        │
        ▼
callMCPToolWithUrlElicitationRetry()   ← 处理 OAuth URL elicitation
        │
        ├── 最多 3 次 URL elicitation 重试
        │
        ▼
callMCPTool()                          ← 核心 JSON-RPC 调用
        │
        ├── client.callTool({name, arguments, _meta}, CallToolResultSchema, {signal, timeout})
        ├── Promise.race([callTool, timeoutPromise])
        │
        ▼
processMCPResult()                     ← 结果转换与截断
        │
        ├── transformMCPResult()
        ├── mcpContentNeedsTruncation()
        ├── persistToolResult() / truncateMcpContentIfNeeded()
        │
        ▼
返回 { data: content, mcpMeta? }
```

### 6.2 callMCPTool 核心实现

```typescript
async function callMCPTool({ client, tool, args, meta, signal, onProgress }) {
  const toolStartTime = Date.now()

  // 30 秒进度日志
  const progressInterval = setInterval(() => {
    logMCPDebug(name, `Tool '${tool}' still running (${elapsed}s elapsed)`)
  }, 30000)

  // 超时保护
  const timeoutMs = getMcpToolTimeoutMs()
  const timeoutPromise = new Promise((_, reject) => {
    setTimeout(() => reject(new Error(`timed out after ${timeoutMs}ms`)), timeoutMs)
  })

  // 实际 JSON-RPC 调用
  const result = await Promise.race([
    client.callTool(
      { name: tool, arguments: args, _meta: meta },
      CallToolResultSchema,
      {
        signal,
        timeout: timeoutMs,
        onprogress: (sdkProgress) => {
          onProgress?.({ type: 'mcp_progress', status: 'progress', ... })
        },
      },
    ),
    timeoutPromise,
  ])

  // 错误检查
  if (result.isError) {
    throw new McpToolCallError(errorDetails)
  }

  // 结果处理
  const content = await processMCPResult(result, tool, name)
  return { content, _meta: result._meta, structuredContent: result.structuredContent }
}
```

### 6.3 Elicitation 处理

MCP 服务器可以通过 Elicitation 请求用户交互（如 OAuth 浏览器授权）：

```typescript
// URL Elicitation — 打开浏览器进行 OAuth
export async function callMCPToolWithUrlElicitationRetry({
  client, tool, args, meta, signal, handleElicitation, callToolFn
}) {
  const MAX_URL_ELICITATION_RETRIES = 3
  for (let attempt = 0; ; attempt++) {
    try {
      return await callToolFn({ client, tool, args, meta, signal })
    } catch (error) {
      if (isUrlElicitationError(error) && attempt < MAX_URL_ELICITATION_RETRIES) {
        // 展示 URL 给用户，等待完成通知
        await handleElicitation(error.elicitationUrl)
        // 重试工具调用
        continue
      }
      throw error
    }
  }
}

// 表单 Elicitation — 在终端中显示表单
client.setRequestHandler(ElicitRequestSchema, async (request) => {
  // 通过 runElicitationHooks → ElicitationDialog 展示
  const result = await handleElicitation(request)
  return result  // { action: 'accept' | 'cancel', content?: ... }
})
```

### 6.4 超时与取消

| 超时类型 | 默认值 | 配置方式 |
|---------|-------|---------|
| 连接超时 | `getConnectionTimeoutMs()` | 环境变量 |
| 工具调用超时 | `getMcpToolTimeoutMs()` | 环境变量 |
| 请求级超时 | `MCP_REQUEST_TIMEOUT_MS` | 常量 |

取消通过 `AbortSignal` 传播：
```typescript
const result = await client.callTool(
  { name, arguments: args },
  CallToolResultSchema,
  { signal: context.abortController.signal, timeout: timeoutMs },
)
```

---

## 7. 结果处理

### 7.1 结果转换

`processMCPResult()` 将 MCP 结果转换为内部格式：

```typescript
export async function processMCPResult(result, tool, name): Promise<MCPToolResult> {
  const { content, type, schema } = await transformMCPResult(result, tool, name)

  // IDE 工具不送模型，直接返回
  if (name === 'ide') return content

  // 检查是否需要截断
  if (!(await mcpContentNeedsTruncation(content))) return content

  // ...大结果处理
}
```

MCP 结果 content 类型：
- `text` → 字符串
- `image` → base64 图片，可能降采样
- `resource` (ResourceLink) → 资源引用
- 二进制数据 → 持久化保存，返回文件路径

### 7.2 大结果处理

当结果超过 token 限制时（`mcpContentNeedsTruncation` = true）：

```typescript
// 1. 尝试保存到文件
const persistId = `mcp-${serverName}-${toolName}-${timestamp}`
const persistResult = await persistToolResult(contentStr, persistId)

if (isPersistError(persistResult)) {
  // 保存失败 → 截断返回
  return truncateMcpContentIfNeeded(content)
}

// 2. 返回读取指令（告诉模型用 FileRead 读取完整内容）
return getLargeOutputInstructions(persistResult.filepath, persistResult.originalSize, formatDescription)
```

### 7.3 图片结果处理

```typescript
// 含图片的大结果 → 截断（不保存到文件，保持可查看性）
if (contentContainsImages(content)) {
  return truncateMcpContentIfNeeded(content)
}

// 图片可能被缩放和降采样
await maybeResizeAndDownsampleImageBuffer(buffer)
```

---

## 8. 工具 Schema 发送到 LLM

### 8.1 toolToAPISchema 转换

每个工具通过 `toolToAPISchema()` 转换为 API 参数格式：

```typescript
// src/utils/api.ts
export async function toolToAPISchema(tool, options): Promise<BetaTool> {
  // 1. 构建基础 schema（session 级缓存）
  const base = {
    name: tool.name,
    description: await tool.prompt({ ... }),
    input_schema: tool.inputJSONSchema || zodToJsonSchema(tool.inputSchema),
  }

  // 2. 可选字段
  if (strictToolsEnabled && tool.strict) base.strict = true
  if (fgtsEnabled) base.eager_input_streaming = true

  // 3. 缓存
  cache.set(cacheKey, base)

  // 4. 每次请求的覆盖层
  const schema = { ...base }
  if (options.deferLoading) schema.defer_loading = true
  if (options.cacheControl) schema.cache_control = options.cacheControl

  return schema
}
```

### 8.2 defer_loading 标记

在 `src/services/api/claude.ts` 中决定哪些工具标记为 defer：

```typescript
const willDefer = (t: Tool) =>
  useToolSearch && (deferredToolNames.has(t.name) || shouldDeferLspTool(t))

// 构建 tool schemas
const toolSchemas = await Promise.all(
  filteredTools.map(tool =>
    toolToAPISchema(tool, {
      // ...
      deferLoading: willDefer(tool),
    }),
  ),
)
```

被标记 `defer_loading: true` 的工具：
- API **不会**将完整 schema 放入模型上下文
- 模型只能通过 `ToolSearchTool` 发现并加载它们

### 8.3 Schema 缓存

Schema 使用 session 级缓存避免 GrowthBook feature flag 翻转导致缓存破坏：

```typescript
// src/utils/toolSchemaCache.ts
const TOOL_SCHEMA_CACHE = new Map<string, CachedSchema>()

// cache key 包含 inputJSONSchema 以支持 StructuredOutput 等同名不同 schema 场景
const cacheKey = tool.inputJSONSchema
  ? `${tool.name}:${JSON.stringify(tool.inputJSONSchema)}`
  : tool.name
```

---

## 9. 资源与命令发现

除工具外，MCP 还支持**资源**和**命令（Prompts）**发现：

```typescript
// 资源发现
const resources = await client.client.request(
  { method: 'resources/list' },
  ListResourcesResultSchema,
)
// 返回 ServerResource[] = Resource & { server: string }

// 命令/Prompt 发现
const prompts = await client.client.request(
  { method: 'prompts/list' },
  ListPromptsResultSchema,
)
```

资源可通过两个专用工具访问：
- `ListMcpResourcesTool` — 列出所有可用资源
- `ReadMcpResourceTool` — 读取特定资源内容

服务器支持 `notifications/tools/list_changed`、`notifications/resources/list_changed`、`notifications/prompts/list_changed` 通知，触发缓存刷新。

---

## 10. 认证与 OAuth

远程 MCP 服务器（SSE/HTTP）支持 OAuth 认证：

```typescript
// src/services/mcp/auth.ts
class ClaudeAuthProvider {
  // 存储/获取 OAuth token（macOS Keychain / 文件存储）
  async tokens(): Promise<OAuthTokens | null>
  async saveTokens(tokens: OAuthTokens): Promise<void>

  // OAuth 流程
  async redirectUrl(): Promise<URL>    // callback URL
  async clientInformation(): Promise<OAuthClientInformation>

  // 自动刷新
  async refreshIfNeeded(): Promise<void>
}
```

**认证流程**：
1. 连接时如果服务器返回 401 → 状态变为 `needs-auth`
2. 生成 `McpAuthTool` — 用户调用时打开浏览器完成 OAuth
3. OAuth callback 收到 token → 存储 → 重新连接
4. Token 过期时自动刷新，或标记 `needs-auth` 等待用户重新授权

**Cross-App Access (XAA)** — 跨应用授权：
- 通过 `settings.xaaIdp` 配置 IdP 连接信息
- 每个服务器通过 `xaa: true` 标记启用

---

## 11. 连接管理 UI (React)

连接管理通过 React Context 提供：

```typescript
// src/services/mcp/MCPConnectionManager.tsx
export function MCPConnectionManager({ children, dynamicMcpConfig, isStrictMcpConfig }) {
  const { reconnectMcpServer, toggleMcpServer } = useManageMCPConnections(
    dynamicMcpConfig, isStrictMcpConfig
  )
  return (
    <MCPConnectionContext.Provider value={{ reconnectMcpServer, toggleMcpServer }}>
      {children}
    </MCPConnectionContext.Provider>
  )
}

// Hooks
export function useMcpReconnect()      // 手动重连特定服务器
export function useMcpToggleEnabled()  // 启用/禁用服务器
```

`useManageMCPConnections` 管理完整生命周期：
- 启动时批量连接所有已配置服务器
- 监听配置变更（config diff → 增量连接/断开）
- 监听 `tools/list_changed` 通知刷新工具缓存
- 注册 elicitation handler
- 自动重连（指数退避）
- 更新 AppState 中的 `mcp.clients`、`mcp.tools`、`mcp.resources`

---

## 12. 关键数据流总结

### 完整时序

```
┌──────────────┐    ┌───────────────┐    ┌──────────────┐    ┌──────────┐
│  Config 层    │    │  Connection 层 │    │  Discovery 层 │    │  API 层   │
└──────┬───────┘    └───────┬───────┘    └──────┬───────┘    └────┬─────┘
       │                    │                   │                  │
  .mcp.json                 │                   │                  │
  settings.json             │                   │                  │
  enterprise-mcp.json       │                   │                  │
  plugin configs            │                   │                  │
       │                    │                   │                  │
       ▼                    │                   │                  │
  getAllMcpConfigs()         │                   │                  │
  ├── scope 合并            │                   │                  │
  ├── 策略过滤              │                   │                  │
  ├── plugin 去重           │                   │                  │
  └── claudeai 去重         │                   │                  │
       │                    │                   │                  │
       ▼                    │                   │                  │
  getMcpToolsCommands       │                   │                  │
  AndResources()            │                   │                  │
       │                    │                   │                  │
       ├───────────────────►│                   │                  │
       │              connectToServer()          │                  │
       │              ├── 创建 Transport          │                  │
       │              ├── Client.connect()        │                  │
       │              └── 获取 capabilities        │                  │
       │                    │                   │                  │
       │                    ├──────────────────►│                  │
       │                    │             fetchToolsForClient()     │
       │                    │             ├── tools/list            │
       │                    │             └── → Tool[] 转换          │
       │                    │                   │                  │
       │                    │             fetchResourcesForClient() │
       │                    │             └── resources/list        │
       │                    │                   │                  │
       │                    │             fetchCommandsForClient()  │
       │                    │             └── prompts/list          │
       │                    │                   │                  │
       │                    │                   ▼                  │
       │                    │            assembleToolPool()         │
       │                    │            ├── getTools() 内置        │
       │                    │            ├── deny 过滤             │
       │                    │            └── 合并去重排序          │
       │                    │                   │                  │
       │                    │                   ├─────────────────►│
       │                    │                   │           toolToAPISchema()
       │                    │                   │           ├── name + desc
       │                    │                   │           ├── input_schema
       │                    │                   │           └── defer_loading?
       │                    │                   │                  │
       │                    │                   │                  ▼
       │                    │                   │          API Request:
       │                    │                   │          { tools: [...] }
       │                    │                   │                  │
       │                    │                   │                  ▼
       │                    │                   │          LLM 返回 tool_use
       │                    │                   │          (可能先 ToolSearch)
       │                    │                   │                  │
       │                    │                   │                  ▼
       │                    │                   │          Tool.call()
       │                    │                   │                  │
       │                    ◄──────────────────────────────────────┤
       │              ensureConnectedClient()                      │
       │              callMCPTool()                                │
       │              ├── client.callTool()                        │
       │              └── processMCPResult()                       │
       │                    │                                      │
       │                    ▼                                      │
       │              MCP Server 执行                               │
       │              └── 返回结果                                  │
       │                    │                                      │
       │                    ├─────────────────────────────────────►│
       │                    │                              tool_result
       │                    │                              → 下一轮 API 调用
```

### 模型视角

```
╔═════════════════════════════════════════════════════════════════╗
║                    模型看到的工具集                               ║
╠═════════════════════════════════════════════════════════════════╣
║                                                                 ║
║  [完整 Schema]                    [仅名称 — deferred]            ║
║  ├── BashTool                    ├── mcp__slack__search          ║
║  ├── FileReadTool                ├── mcp__slack__send_message    ║
║  ├── FileEditTool                ├── mcp__github__create_pr      ║
║  ├── FileWriteTool               ├── mcp__postgres__query        ║
║  ├── GlobTool                    ├── mcp__jira__create_issue     ║
║  ├── GrepTool                    └── ...                        ║
║  ├── WebFetchTool                                               ║
║  ├── AgentTool                   ← 通过 ToolSearchTool 按需加载  ║
║  ├── ToolSearchTool ─────────────────────────────┐              ║
║  ├── ...                                         │              ║
║  └── [alwaysLoad MCP 工具]                       │              ║
║                                                  │              ║
║  ToolSearch("slack")                             │              ║
║       │                                          │              ║
║       ▼                                          │              ║
║  tool_reference: mcp__slack__search ◄────────────┘              ║
║  tool_reference: mcp__slack__send_message                       ║
║       │                                                         ║
║       ▼  (API 自动注入完整 schema)                               ║
║  模型现在可以调用 mcp__slack__search                              ║
║                                                                 ║
╚═════════════════════════════════════════════════════════════════╝
```

---

> **核心设计理念**：Claude Code 通过多层缓存（连接缓存 + 工具缓存 + Schema 缓存）、延迟加载（Tool Search / defer_loading）、以及 prompt cache 稳定性优化（排序策略），在支持大量 MCP 工具的同时保持低延迟和高效的 token 利用。
