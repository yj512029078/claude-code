---
title: Hooks 系统
aliases: [Hooks 系统, Hooks System]
series: Claude Code 架构解析
category: 扩展生态
order: 13
tags:
  - claude-code
  - hooks
  - lifecycle
  - event-system
  - extensibility
date: 2026-04-04
---

# Hooks 系统详细技术文档

## 1. 概述

Hooks 系统是 Claude Code 的**生命周期扩展机制**，允许用户在工具执行、权限决策、Session 生命周期等关键节点注入自定义逻辑。它支持 6 种执行模式、27 种事件类型，并提供丰富的 JSON 结构化输出控制工具行为。

### 核心能力

- **工具守门**：在工具执行前后注入验证逻辑 (PreToolUse / PostToolUse)
- **权限决策**：通过 Hook 自动审批/拒绝/修改工具调用
- **输入改写**：动态修改工具输入参数
- **流程控制**：阻止 agent 停止 (Stop hook)、注入额外上下文
- **环境感知**：监听文件变更、配置变更、Worktree 操作
- **异步后台处理**：Hook 可后台执行，完成后唤醒模型

> **重要区分**：`src/hooks/` 目录下主要是 **React hooks** (UI 层的 `use*` 函数)。Hooks *系统*的核心实现在 `src/utils/hooks.ts` 和 `src/utils/hooks/` 目录下。

---

## 2. 架构总览

```
┌─────────────────────────────────────────────────────────────────┐
│                        配置来源                                  │
│                                                                  │
│  settings.json     session hooks      plugins/skills    SDK      │
│  (user/project/    (in-memory,        (registerHook     callback │
│   local/managed)    skill/agent 注册)   Callbacks)       注册)   │
│       │                 │                  │               │     │
│       ▼                 ▼                  ▼               ▼     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              hooksConfigSnapshot (合并快照)                │   │
│  │  captureHooksConfigSnapshot() → 启动时冻结                │   │
│  │  updateHooksConfigSnapshot() → settings 变更时更新        │   │
│  │  + getRegisteredHooks() → plugin/SDK 回调                 │   │
│  │  + getSessionHooks() → per-agent session hooks            │   │
│  └──────────────────────────────────────────────────────────┘   │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                      getHooksConfig()                            │
│  合并: snapshot + registeredHooks + sessionHooks + functionHooks │
│  策略过滤: disableAllHooks / allowManagedHooksOnly              │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    getMatchingHooks()                             │
│  1. 按 HookEvent 过滤                                           │
│  2. matcher 模式匹配 (工具名 / glob)                             │
│  3. if 条件评估 (权限规则语法)                                    │
│  4. 去重                                                        │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                      executeHooks()                              │
│  async generator — 并行执行所有匹配 hooks                        │
│                                                                  │
│  ┌─────────┐ ┌─────────┐ ┌────────┐ ┌───────┐ ┌────────┐      │
│  │ command  │ │  http   │ │ prompt │ │ agent │ │callback│      │
│  │ (shell)  │ │ (POST)  │ │ (LLM)  │ │(query)│ │ (SDK)  │      │
│  └────┬─────┘ └────┬────┘ └───┬────┘ └───┬───┘ └───┬────┘      │
│       │            │          │          │         │             │
│       ▼            ▼          ▼          ▼         ▼             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              processHookJSONOutput()                       │   │
│  │  解析 JSON → HookResult                                   │   │
│  │  - permissionBehavior (allow/deny/ask)                     │   │
│  │  - updatedInput (修改工具输入)                              │   │
│  │  - blockingError (阻止执行)                                │   │
│  │  - additionalContext (注入上下文)                           │   │
│  │  - preventContinuation (阻止停止)                          │   │
│  └──────────────────────────────────────────────────────────┘   │
│       │                                                          │
│       ▼                                                          │
│  yield AggregatedHookResult  (聚合多个 hook 结果)                │
│  权限合并: deny > ask > allow                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. Hook 配置格式

### 3.1 Settings 中的 Hook 定义

Hooks 在 `settings.json` (user/project/local/managed) 中配置：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "command",
            "command": "python validate.py",
            "timeout": 30,
            "if": "Write(*.ts)"
          }
        ]
      },
      {
        "matcher": "",
        "hooks": [
          {
            "type": "http",
            "url": "https://hooks.example.com/pre-tool",
            "headers": { "Authorization": "Bearer $MY_TOKEN" },
            "allowedEnvVars": ["MY_TOKEN"]
          }
        ]
      }
    ],
    "Stop": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "agent",
            "prompt": "Verify that all tests pass. $ARGUMENTS",
            "model": "claude-sonnet-4-6",
            "timeout": 60
          }
        ]
      }
    ]
  }
}
```

### 3.2 配置 Schema

```typescript
// HooksSettings: 事件 → Matcher 数组
type HooksSettings = Partial<Record<HookEvent, HookMatcher[]>>

// HookMatcher: 模式匹配 + hooks 列表
type HookMatcher = {
  matcher?: string     // 工具名/模式 (如 "Write", "Bash|Read")
  hooks: HookCommand[] // 该 matcher 下的 hooks
}

// HookCommand: 4 种可持久化的 hook 类型 (discriminated union)
type HookCommand = BashCommandHook | PromptHook | AgentHook | HttpHook
```

### 3.3 四种 Hook 类型

#### Command Hook (Shell)

```typescript
type BashCommandHook = {
  type: 'command'
  command: string           // Shell 命令
  if?: string               // 权限规则语法过滤 (如 "Bash(git *)")
  shell?: 'bash' | 'powershell'
  timeout?: number          // 秒
  statusMessage?: string    // spinner 显示文本
  once?: boolean            // 一次性执行后移除
  async?: boolean           // 后台运行，不阻塞
  asyncRewake?: boolean     // 后台运行，exit code 2 时唤醒模型
}
```

#### HTTP Hook

```typescript
type HttpHook = {
  type: 'http'
  url: string               // POST 目标 URL
  if?: string
  timeout?: number          // 秒
  headers?: Record<string, string>  // 支持 $VAR_NAME 插值
  allowedEnvVars?: string[] // 允许插值的环境变量白名单
  statusMessage?: string
  once?: boolean
}
```

#### Prompt Hook (LLM 单轮)

```typescript
type PromptHook = {
  type: 'prompt'
  prompt: string            // $ARGUMENTS 占位符替换为 hook 输入 JSON
  if?: string
  timeout?: number          // 秒 (默认 30s)
  model?: string            // 模型 (默认 small fast model / Haiku)
  statusMessage?: string
  once?: boolean
}
```

#### Agent Hook (LLM 多轮)

```typescript
type AgentHook = {
  type: 'agent'
  prompt: string            // $ARGUMENTS 占位符
  if?: string
  timeout?: number          // 秒 (默认 60s)
  model?: string            // 模型 (默认 Haiku)
  statusMessage?: string
  once?: boolean
}
```

### 3.4 非持久化 Hook 类型 (运行时)

| 类型 | 注册方式 | 说明 |
|------|---------|------|
| **callback** | `registerHookCallbacks()` (Bootstrap/SDK) | SDK 回调，通过 control_request 发送到 SDK 消费者 |
| **function** | `addFunctionHook()` (Session) | 进程内 TypeScript 回调，用于结构化输出验证等 |

---

## 4. 27 种 Hook 事件

### 4.1 工具生命周期

| 事件 | 触发时机 | matcher 含义 | 可影响 |
|------|---------|-------------|--------|
| `PreToolUse` | 工具执行前 | 工具名 | 权限决策、修改输入、阻止执行 |
| `PostToolUse` | 工具执行后 | 工具名 | 注入上下文、修改 MCP 输出、阻止后续 |
| `PostToolUseFailure` | 工具执行失败后 | 工具名 | 注入上下文 |

### 4.2 权限

| 事件 | 触发时机 | 可影响 |
|------|---------|--------|
| `PermissionRequest` | 权限需要人工决策时 | allow/deny + 修改输入 + 更新权限规则 |
| `PermissionDenied` | 权限被拒绝后 | retry (重试) |

### 4.3 Session 生命周期

| 事件 | 触发时机 | 可影响 |
|------|---------|--------|
| `SessionStart` | Session 启动 | 注入上下文、初始用户消息、watchPaths |
| `SessionEnd` | Session 结束 | (仅通知) |
| `Setup` | 初始化/维护 | 注入上下文 |
| `Stop` | Agent 准备停止时 | **阻止停止** (continue: false) |
| `StopFailure` | Agent 因错误停止时 | (仅通知) |

### 4.4 用户交互

| 事件 | 触发时机 | 可影响 |
|------|---------|--------|
| `UserPromptSubmit` | 用户提交提示前 | 注入上下文 |
| `Notification` | 系统通知 | 注入上下文 |

### 4.5 Agent / Team

| 事件 | 触发时机 | 可影响 |
|------|---------|--------|
| `SubagentStart` | 子 Agent 启动 | 注入上下文 |
| `SubagentStop` | 子 Agent 停止 | 注入上下文 |
| `TeammateIdle` | 团队成员空闲 | (仅通知) |
| `TaskCreated` | 任务创建 | (仅通知) |
| `TaskCompleted` | 任务完成 | (仅通知) |

### 4.6 Compaction

| 事件 | 触发时机 | 可影响 |
|------|---------|--------|
| `PreCompact` | Compaction 前 | (仅通知) |
| `PostCompact` | Compaction 后 | (仅通知) |

### 4.7 MCP Elicitation

| 事件 | 触发时机 | 可影响 |
|------|---------|--------|
| `Elicitation` | MCP 请求用户输入 | 自动 accept/decline/cancel |
| `ElicitationResult` | 用户回复 elicitation 后 | 覆盖 action/content |

### 4.8 配置与环境

| 事件 | 触发时机 | 可影响 |
|------|---------|--------|
| `ConfigChange` | 设置文件变更 | (仅通知) |
| `InstructionsLoaded` | Memory/Instructions 加载 | (仅通知) |
| `CwdChanged` | 工作目录变更 | watchPaths |
| `FileChanged` | 监视的文件变更 | watchPaths |
| `WorktreeCreate` | Git Worktree 创建 | worktreePath |
| `WorktreeRemove` | Git Worktree 移除 | (仅通知) |

---

## 5. Hook 输入 (HookInput)

每种事件有特定的输入结构，所有都继承自 `BaseHookInput`：

```typescript
type BaseHookInput = {
  session_id: string
  transcript_path: string
  cwd: string
  permission_mode?: string
  agent_id?: string      // 在子 agent 中时存在
  agent_type?: string    // Agent 类型名
}
```

**关键事件输入示例：**

```typescript
// PreToolUse
{
  ...base,
  hook_event_name: 'PreToolUse',
  tool_name: 'Write',
  tool_input: { file_path: '/src/main.ts', content: '...' },
  tool_use_id: 'toolu_abc123'
}

// PostToolUse
{
  ...base,
  hook_event_name: 'PostToolUse',
  tool_name: 'Bash',
  tool_input: { command: 'npm test' },
  tool_response: { stdout: 'All tests passed', exit_code: 0 },
  tool_use_id: 'toolu_def456'
}

// Stop
{
  ...base,
  hook_event_name: 'Stop',
  stop_hook_active: true,
  last_assistant_message: '...'
}
```

**Command Hook** 的输入通过环境变量和 stdin JSON 传递：
- `CLAUDE_HOOK_EVENT` — 事件名
- `CLAUDE_HOOK_INPUT` — 输入 JSON 字符串 (如果不超长)
- `CLAUDE_ENV_FILE` — 包含环境变量的文件路径 (用于超长输入)
- stdin → 完整 JSON 输入

---

## 6. Hook 输出 (HookJSONOutput)

### 6.1 同步输出

Hook 通过 stdout (Command)、HTTP response body (HTTP)、或返回值 (Callback/Function) 输出 JSON：

```typescript
type SyncHookJSONOutput = {
  // 全局控制
  continue?: boolean          // false = 阻止后续执行 (Stop hook: 阻止 agent 停止)
  suppressOutput?: boolean    // true = 隐藏 stdout 不进入 transcript
  stopReason?: string         // continue=false 时的停止原因
  decision?: 'approve' | 'block'  // 传统格式 (映射为 allow/deny)
  reason?: string             // 决策理由
  systemMessage?: string      // 系统警告消息

  // 事件特定输出
  hookSpecificOutput?: {
    hookEventName: 'PreToolUse'
    permissionDecision?: 'allow' | 'deny' | 'ask'
    permissionDecisionReason?: string
    updatedInput?: Record<string, unknown>   // 修改工具输入
    additionalContext?: string               // 注入上下文到对话
  } | {
    hookEventName: 'PermissionRequest'
    decision: {
      behavior: 'allow'
      updatedInput?: Record<string, unknown>
      updatedPermissions?: PermissionUpdate[]  // 持久化权限规则
    } | {
      behavior: 'deny'
      message?: string
      interrupt?: boolean
    }
  } | {
    hookEventName: 'PostToolUse'
    additionalContext?: string
    updatedMCPToolOutput?: unknown  // 修改 MCP 工具输出
  } | {
    hookEventName: 'SessionStart'
    additionalContext?: string
    initialUserMessage?: string     // 注入初始用户消息
    watchPaths?: string[]           // FileChanged 监视路径
  } | {
    hookEventName: 'Elicitation'
    action?: 'accept' | 'decline' | 'cancel'
    content?: Record<string, unknown>
  }
  // ... 更多事件特定输出
}
```

### 6.2 异步输出

Command Hook 可以声明异步执行：

```json
{"async": true, "asyncTimeout": 15000}
```

此时 Hook 进程移入后台，不阻塞工具执行。完成后：
- 输出扫描最后一行同步 JSON (非 `async`)
- 结果作为 `async_hook_response` attachment 注入对话
- `asyncRewake: true` 时 exit code 2 会唤醒模型

### 6.3 非 JSON 输出 (Command Hook)

Command Hook 也支持纯 exit code 驱动：

| Exit Code | 语义 | 效果 |
|-----------|------|------|
| 0 | 成功 | Hook 结果为 success |
| 2 | 阻塞错误 | 生成 `blockingError`，对应 PreToolUse → deny |
| 其他非零 | 非阻塞错误 | 生成 `non_blocking_error` attachment |

---

## 7. 执行引擎

### 7.1 核心执行流程

```
executePreToolHooks(toolName, toolUseID, input, ...)
  │
  ├── createBaseHookInput() + 事件特定字段 → HookInput
  │
  ├── getMatchingHooks(hookEvent, toolName, input)
  │     ├── getHooksConfig() → 合并所有来源
  │     ├── 按 matcher 模式匹配过滤
  │     ├── 按 if 条件 (权限规则语法) 过滤
  │     └── 去重
  │
  └── yield* executeHooks(matchingHooks, input, ...)
        │
        ├── 并行执行所有 hooks (Promise.all)
        │     ├── command → execCommandHook()
        │     ├── http    → execHttpHook()
        │     ├── prompt  → execPromptHook()
        │     ├── agent   → execAgentHook()
        │     ├── callback→ executeHookCallback()
        │     └── function→ executeFunctionHook()
        │
        ├── 每个 hook 完成后 yield HookResult
        │     - emitHookStarted / emitHookResponse
        │     - parseHookOutput → processHookJSONOutput
        │
        └── 最终 yield AggregatedHookResult
              - 权限合并: deny > ask > allow
              - updatedInput 传播
              - blockingErrors 收集
              - additionalContexts 合并
```

### 7.2 六种执行模式详解

#### Command Hook (`execCommandHook`)

```
spawn shell → 传入环境变量 + stdin JSON → 读取 stdout → parseHookOutput
                                                             │
                                              ┌──────────────┴──────────────┐
                                              │                             │
                                         JSON 输出                    exit code
                                   processHookJSONOutput          0/2/other 映射
```

- Shell 选择：`bash` → `$SHELL` (bash/zsh/sh)，`powershell` → `pwsh`
- 环境变量注入：`CLAUDE_HOOK_EVENT`, `CLAUDE_HOOK_INPUT`, `CLAUDE_TOOL_USE_ID`, `CLAUDE_TOOL_USE_INPUT`, `CLAUDE_SESSION_ID`, `CLAUDE_TRANSCRIPT_PATH` 等
- stdout 可包含混合内容：Prompt Request JSON + Hook JSON（最后一个非 async JSON 行生效）
- async hook → `registerPendingAsyncHook()` → 后台监控

#### HTTP Hook (`execHttpHook`)

```
POST JSON body → 读取 response body → parseHttpHookOutput → processHookJSONOutput
```

- **SSRF 防护**：`ssrfGuardedLookup` DNS 检查，阻止内网地址
- **URL 白名单**：`allowedHttpHookUrls` 设置项 (模式匹配)
- **Header 环境变量插值**：`$VAR_NAME` / `${VAR_NAME}` 语法，仅 `allowedEnvVars` 中的变量生效
- **Header 注入防护**：`sanitizeHeaderValue` 移除 CR/LF/NUL
- **沙箱代理**：启用沙箱时通过 `SandboxManager` 代理网络请求
- **限制**：`SessionStart` / `Setup` 事件跳过 HTTP hooks（防止 headless 死锁）

#### Prompt Hook (`execPromptHook`)

```
替换 $ARGUMENTS → queryModelWithoutStreaming() → 解析 {ok, reason} → 映射到 HookResult
```

- 使用 `queryModelWithoutStreaming` 单轮调用
- 系统提示要求返回 `{"ok": true}` 或 `{"ok": false, "reason": "..."}`
- `ok: false` → blocking error
- 默认超时 30s，默认模型为 small fast model

#### Agent Hook (`execAgentHook`)

```
替换 $ARGUMENTS → query() 多轮循环 → StructuredOutput tool 收集 {ok, reason}
```

- 使用完整 `query()` 循环（包含工具调用能力）
- 注入 `StructuredOutput` 工具强制结构化输出
- `registerStructuredOutputEnforcement` 注册 function hook 确保 agent 调用结构化输出工具
- 最多 30 轮，默认超时 60s
- Agent 使用的工具被过滤（排除 `ALL_AGENT_DISALLOWED_TOOLS`）

#### SDK Callback Hook (`executeHookCallback`)

```
hook.callback(input, toolUseID, signal) → HookJSONOutput → processHookJSONOutput
```

- 由 `registerHookCallbacks()` 在 Bootstrap 时注册（Plugin/SDK）
- SDK 消费者通过 `initialize` 请求的 `hooks` 字段注册
- 实际执行通过 `control_request { subtype: 'hook_callback' }` 发送到 SDK 进程

#### Function Hook (`executeFunctionHook`)

```
hook.callback(messages, signal) → boolean → true=pass, false=blocking error
```

- 纯 TypeScript 回调，进程内执行
- 用于结构化输出验证、plan 执行验证等内部场景
- `errorMessage` 字段定义失败时的阻塞消息

---

## 8. 权限交互

### 8.1 PreToolUse Hook 权限决策

PreToolUse Hook 可以返回权限决策，但 **Hook allow 不能绕过规则引擎的 deny/ask 规则**：

```
PreToolUse hook → permissionDecision: 'allow'
       │
       ▼
resolveHookPermissionDecision()
       │
       ├── checkRuleBasedPermissions()
       │     ├── null (无规则) → 接受 hook allow ✓
       │     ├── deny → deny 覆盖 hook allow ✗
       │     └── ask → 仍需用户/SDK 确认
       │
       ├── requiresUserInteraction?
       │     └── updatedInput 存在 → 视为交互已满足
       │
       └── requireCanUseTool?
             └── 仍需调用 canUseTool()
```

**权限合并规则（多 hook 并行时）**：

```
deny > ask > allow
```

即任何一个 hook deny 都会覆盖其他 hook 的 allow。

### 8.2 PermissionRequest Hook

PermissionRequest Hook 在权限系统判定需要"ask"（需要人工决策）时触发。与 SDK `can_use_tool` 回调**并行竞速**：

```
hasPermissionsToUseTool() → ask
       │
       ├── executePermissionRequestHooks() ──┐
       │                                      │
       └── SDK can_use_tool prompt ───────────┤
                                               │
                          Promise.race ◄───────┘
                               │
                    先返回者的决策生效
```

Hook 可以返回：
- `{ behavior: 'allow', updatedInput?, updatedPermissions? }` — 批准 + 可选更新权限规则
- `{ behavior: 'deny', message?, interrupt? }` — 拒绝

---

## 9. `if` 条件过滤

`if` 字段使用**权限规则语法**在 hook 匹配阶段过滤，避免为不相关的调用启动 hook 进程：

```json
{
  "type": "command",
  "command": "check-git-safety.sh",
  "if": "Bash(git *)"
}
```

语法：`ToolName(pattern)` — 仅当工具名和输入匹配时运行。

示例：
- `"Bash(git *)"` — 仅 Bash 工具执行 git 命令时
- `"Write(*.ts)"` — 仅写入 .ts 文件时
- `"Read(src/**)"` — 仅读取 src 目录下文件时

---

## 10. 异步 Hook 机制

### 10.1 声明异步

Command Hook 在 stdout 输出：

```json
{"async": true, "asyncTimeout": 15000}
```

### 10.2 执行流程

```
Hook 返回 {async:true}
       │
       ▼
registerPendingAsyncHook()  ← AsyncHookRegistry.ts
  - 记录 processId, hookId, shellCommand
  - 启动 progressInterval (1s 轮询输出)
  - 默认超时 15s
       │
       ▼
(工具执行继续，不等待 hook)
       │
       ▼
(每轮 query 循环检查)
checkForAsyncHookResponses()
  - 扫描已完成的后台 hook
  - 解析 stdout 中的同步 JSON (非 async 行)
  - 生成 async_hook_response attachment
       │
       ▼
removeDeliveredAsyncHooks()
  - 清理已投递的 hook
```

### 10.3 asyncRewake 模式

`asyncRewake: true` 是 `async` 的增强模式：
- 后台执行，不阻塞
- 如果 exit code 为 2（blocking error），通过 `enqueuePendingNotification` 唤醒模型
- 不使用 AsyncHookRegistry（避免 attachment 重复）

---

## 11. Hook 事件广播 (hookEvents)

Hook 执行生命周期事件通过 `hookEvents.ts` 广播给 SDK 消费者：

```typescript
// 事件类型
type HookExecutionEvent =
  | HookStartedEvent    // hook 开始执行
  | HookProgressEvent   // hook 输出进度 (定时轮询)
  | HookResponseEvent   // hook 执行完成

// 注册处理器 (print.ts 在 stream-json 模式注册)
registerHookEventHandler((event) => {
  output.enqueue({
    type: 'system',
    subtype: `hook_${event.type}`,  // hook_started / hook_progress / hook_response
    ...event,
    uuid: randomUUID(),
    session_id: getSessionId()
  })
})
```

**选择性发射**：
- `SessionStart` / `Setup` 事件的 hook → **始终发射** (低噪)
- 其他事件 → 仅当 `setAllHookEventsEnabled(true)` 时发射 (SDK `includeHookEvents` 或 remote 模式)

**缓冲**：Handler 未注册时，事件缓冲在 `pendingEvents` 中（上限 100），Handler 注册后立即投递。

---

## 12. 超时管理

| Hook 类型 | 默认超时 | 来源 |
|-----------|---------|------|
| Command | 10 分钟 | `TOOL_HOOK_EXECUTION_TIMEOUT_MS` |
| HTTP | 10 分钟 | `DEFAULT_HTTP_HOOK_TIMEOUT_MS` |
| Prompt | 30 秒 | `execPromptHook` 硬编码 |
| Agent | 60 秒 | `execAgentHook` 硬编码 |
| SessionEnd | 1.5 秒 | `getSessionEndHookTimeoutMs()` + `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS` env |
| Async | 15 秒 | `asyncTimeout` 或默认 |

每个 hook 可通过 `timeout` 字段覆盖默认值（秒）。超时通过 `createCombinedAbortSignal` 与父级 AbortSignal 合并。

---

## 13. Session Hooks (运行时注册)

### 13.1 来源

- **Skills**：`registerSkillHooks()` 解析 skill frontmatter 中的 hooks 定义
- **Agents**：`registerFrontmatterHooks()` 解析 agent 定义中的 hooks
- **内部功能**：`addFunctionHook()` 注册 TypeScript 回调（如结构化输出验证）

### 13.2 存储

```typescript
// SessionHooksState: Map<agentId, SessionStore>
type SessionStore = {
  hooks: { [event in HookEvent]?: SessionHookMatcher[] }
}

type SessionHookMatcher = {
  matcher: string
  skillRoot?: string
  hooks: Array<{
    hook: HookCommand | FunctionHook
    onHookSuccess?: OnHookSuccess
  }>
}
```

使用 `Map` 而非 `Record` — 避免高并发下的 O(N²) 复制开销。`addFunctionHook` 的 `.set()` 是 O(1)，且不触发 store listener。

### 13.3 once 语义

`once: true` 的 hook 在首次成功执行后自动移除：

```typescript
onHookSuccess: (hook) => {
  removeSessionHook(setAppState, sessionId, event, matcher, hook)
}
```

---

## 14. 策略控制

### 14.1 管理策略

| 设置项 | 效果 |
|--------|------|
| `disableAllHooks` | 禁用所有 hooks（包括 managed） |
| `allowManagedHooksOnly` | 仅允许 managed settings 中定义的 hooks |
| `strictPluginOnlyCustomization` | 严格限制非 plugin hooks |
| `allowedHttpHookUrls` | HTTP hook URL 白名单 (支持通配符) |
| `httpHookAllowedEnvVars` | HTTP hook header 中允许的环境变量 |

### 14.2 快照机制

启动时 `captureHooksConfigSnapshot()` 冻结 hooks 配置。Settings 变更时通过 `updateHooksConfigSnapshot()` 更新。这确保了：
- Hook 配置在 session 内一致
- 策略限制不被运行时修改绕过

---

## 15. 工具执行中的 Hook 集成

### 15.1 完整工具执行路径

```
toolExecution.ts
  │
  ├── runPreToolUseHooks()                    ← toolHooks.ts
  │     └── executePreToolHooks()             ← hooks.ts
  │           ├── yield hookPermissionResult  → 权限决策
  │           ├── yield hookUpdatedInput      → 修改输入
  │           ├── yield preventContinuation   → 阻止继续
  │           └── yield additionalContext     → 注入上下文
  │
  ├── resolveHookPermissionDecision()         ← toolHooks.ts
  │     └── hook allow + 规则引擎检查 → 最终权限
  │
  ├── canUseTool() (如果需要)
  │
  ├── tool.call(input) → 工具执行
  │
  ├── runPostToolUseHooks()                   ← toolHooks.ts
  │     └── executePostToolHooks()
  │           ├── blockingError → attachment
  │           ├── preventContinuation → stop
  │           ├── additionalContext → attachment
  │           └── updatedMCPToolOutput → 修改输出
  │
  └── (失败时) runPostToolUseFailureHooks()
```

### 15.2 Stop Hook 特殊路径

```
query.ts → agent 准备停止
  │
  ├── handleStopHooks()                       ← query/stopHooks.ts
  │     └── executeStopHooks()                ← hooks.ts
  │           └── 输出: {continue: false}
  │                 → preventContinuation = true
  │                 → agent 继续运行
  │
  └── 检查 stopHookResult.preventContinuation
        ├── true  → 不停止，继续 agentic loop
        └── false → 正常停止
```

---

## 16. Prompt Request 协议

Command Hook 可以在执行过程中**向用户请求输入**（交互式选择）：

```json
// Hook stdout 输出 (prompt request)
{
  "prompt": "request-id-123",
  "message": "Choose a test framework:",
  "options": [
    {"key": "jest", "label": "Jest", "description": "Meta's testing framework"},
    {"key": "vitest", "label": "Vitest", "description": "Vite-native testing"}
  ]
}
```

终端 UI 显示选择对话框，用户选择后通过 stdin 返回：

```json
{"prompt_response": "request-id-123", "selected": "vitest"}
```

Hook 进程继续执行，可以基于用户选择输出最终 JSON 结果。

---

## 17. 关键文件索引

| 文件 | 角色 |
|------|------|
| `src/utils/hooks.ts` | 核心引擎 (~5k 行): 所有 `execute*Hooks` 函数、`executeHooks` generator、`processHookJSONOutput` |
| `src/schemas/hooks.ts` | Zod schema 定义: `HookCommandSchema`, `HookMatcherSchema`, `HooksSchema` |
| `src/types/hooks.ts` | TypeScript 类型: `HookResult`, `AggregatedHookResult`, `HookCallback`, `HookProgress` |
| `src/utils/hooks/hookEvents.ts` | 事件广播: `registerHookEventHandler`, `emitHookStarted/Progress/Response` |
| `src/utils/hooks/hooksConfigSnapshot.ts` | 配置快照: `captureHooksConfigSnapshot`, 策略过滤 |
| `src/utils/hooks/hooksSettings.ts` | 配置合并: `getAllHooks`, `getHooksForEvent`, UI 展示 |
| `src/utils/hooks/sessionHooks.ts` | Session hooks: `addSessionHook`, `addFunctionHook` |
| `src/utils/hooks/execHttpHook.ts` | HTTP hook 执行: POST + SSRF 防护 + URL 白名单 |
| `src/utils/hooks/execPromptHook.ts` | Prompt hook: LLM 单轮调用 |
| `src/utils/hooks/execAgentHook.ts` | Agent hook: LLM 多轮 query |
| `src/utils/hooks/AsyncHookRegistry.ts` | 异步 hook 注册表 |
| `src/utils/hooks/registerSkillHooks.ts` | Skill → session hook 注册 |
| `src/utils/hooks/registerFrontmatterHooks.ts` | Agent/Skill frontmatter → hooks |
| `src/utils/hooks/hookHelpers.ts` | 共享工具: schema, `$ARGUMENTS` 替换, 结构化输出 |
| `src/services/tools/toolHooks.ts` | 工具层集成: `runPreToolUseHooks`, `resolveHookPermissionDecision` |
| `src/services/tools/toolExecution.ts` | 工具执行: 调用 hook 的实际入口 |
| `src/query/stopHooks.ts` | Stop hook 编排: `handleStopHooks` |
| `src/entrypoints/sdk/coreSchemas.ts` | SDK 类型: 所有 HookInput/Output Zod schemas |

---

## 18. 关键设计决策

### 18.1 并行执行 + deny 优先

**决策：** 同一事件的多个 hook 并行执行，权限决策按 deny > ask > allow 合并。

**原因：** 并行最大化速度，deny 优先确保安全——一个 hook 的批准不能覆盖另一个的拒绝。

### 18.2 Hook allow 不绕过规则引擎

**决策：** PreToolUse hook 返回 `allow` 后，仍需通过 `checkRuleBasedPermissions()`。

**原因：** 防止第三方 hook（如 MCP plugin 注册的 hook）绕过用户设置的安全规则。规则引擎的 deny 是不可绕过的。

### 18.3 六种执行模式

**决策：** 支持 command、http、prompt、agent、callback、function 六种执行模式。

**原因：** 不同场景需要不同能力——command 最灵活（任意脚本）、http 最适合集成外部系统、prompt/agent 利用 LLM 做智能验证、callback 适合 SDK 集成、function 适合进程内快速校验。

### 18.4 if 条件预过滤

**决策：** 在匹配阶段用权限规则语法过滤，而非在 hook 内部自行判断。

**原因：** 避免为不相关的工具调用启动进程/网络请求。`if: "Bash(git *)"` 可以避免所有非 git 的 Bash 调用触发 hook。

### 18.5 SessionStart/Setup 跳过 HTTP

**决策：** `SessionStart` 和 `Setup` 事件不执行 HTTP hooks。

**原因：** 这些事件在 headless 模式的初始化阶段触发，此时网络/代理可能未就绪，HTTP 调用会导致死锁。
