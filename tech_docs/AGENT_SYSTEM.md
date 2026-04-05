---
title: Claude Code Agent 管理系统
aliases: [Agent 系统, Agent System]
series: Claude Code 架构解析
category: 工具与Agent
order: 9
tags:
  - claude-code
  - agent
  - multi-agent
  - swarm
  - coordinator
date: 2026-04-04
---

# Claude Code Agent 管理系统深度技术文档

## 1. 概览

Claude Code 的 Agent 系统支持**多层级、多模式**的 Agent 编排，从简单的同步子 Agent 到复杂的多 Agent 团队协作（Swarm）。

```
┌──────────────────────────────────────────────────────┐
│                  用户 (REPL/SDK)                      │
└────────────────────────┬─────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────┐
│               主 Agent (Main Loop)                    │
│  QueryEngine → query() → 模型决定调用 AgentTool       │
└────┬───────────┬───────────┬────────────┬────────────┘
     │           │           │            │
  同步子Agent  异步子Agent  Teammate    远程Agent
  (前台阻塞)   (后台运行)  (Swarm成员)  (云端执行)
```

核心组件：

| 组件 | 文件 | 角色 |
|------|------|------|
| **AgentTool** | `tools/AgentTool/AgentTool.tsx` | 创建和管理子 Agent |
| **runAgent** | `tools/AgentTool/runAgent.ts` | 实际执行 Agent 推理循环 |
| **Task** | `Task.ts` | 任务状态基础定义 |
| **TaskOutputTool** | `tools/TaskOutputTool/` | 读取 Agent 输出 |
| **TaskStopTool** | `tools/TaskStopTool/` | 停止运行中的 Agent |
| **SendMessageTool** | `tools/SendMessageTool/` | Agent 间通信 |
| **TeamCreateTool** | `tools/TeamCreateTool/` | 创建 Agent 团队 |
| **coordinatorMode** | `coordinator/coordinatorMode.ts` | 协调器模式 |

---

## 2. 什么时候创建子 Agent

### 2.1 触发条件

子 Agent 的创建**完全由模型决定**。当主模型（或父 Agent）在推理过程中认为需要并行处理或委托任务时，它会调用 `AgentTool`：

```json
{
  "tool": "Agent",
  "input": {
    "prompt": "Search for all TODO comments in the codebase and categorize them",
    "description": "TODO scanner",
    "subagent_type": "general-purpose",
    "run_in_background": true
  }
}
```

### 2.2 模型的决策依据（System Prompt 指导）

`AgentTool/prompt.ts` 中给模型的指引：

```
Agent 工具用于：
- 需要并行处理多个独立任务时
- 任务可以在后台完成、不需要立即结果时
- 需要隔离的上下文环境时（如 worktree）
- 协调器模式下分派工作给 worker

可用的 Agent 类型：
- general-purpose: 通用 Agent
- 自定义 Agent: 从 .claude/agents/ 或内建 Agent
```

### 2.3 自动异步的场景

某些情况下，Agent 会被**强制异步运行**：

```typescript
shouldRunAsync = 
  run_in_background === true        // 模型明确要求后台
  || agent.background === true      // Agent 定义标记为后台
  || isCoordinatorMode()            // 协调器模式下 worker 总是异步
  || forceAsync (fork gate)         // fork 子 Agent 强制异步
  || isKairosMode()                 // KAIROS 主动模式
  || isProactiveMode()              // 主动模式
```

---

## 3. Agent 定义系统

### 3.1 AgentDefinition 类型

```typescript
type AgentDefinition = BuiltInAgentDefinition | CustomAgentDefinition

type CustomAgentDefinition = {
  agentType: string              // 标识符 (如 "researcher", "code-reviewer")
  source: 'user' | 'project' | 'plugin' | 'flag' | 'managed'
  getSystemPrompt(): string      // 系统提示词
  tools?: string[]               // 限制可用工具
  disallowedTools?: string[]     // 禁止的工具
  maxTurns?: number              // 最大轮次
  background?: boolean           // 是否默认后台运行
  isolation?: 'worktree' | 'remote'  // 隔离模式
  memory?: boolean               // 是否启用记忆
  hooks?: HooksSettings          // 关联钩子
  requiredMcpServers?: string[]  // 必需的 MCP 服务器
  color?: AgentColorName         // UI 颜色
}
```

### 3.2 定义来源与优先级

```
加载优先级 (后者覆盖前者同名 Agent):

  built-in (内建)
    → plugin (插件)
      → user (~/.claude/agents/)
        → project (.claude/agents/)
          → flag (Feature Flag)
            → managed (组织策略)

// 同一 agentType 的定义，后加载的覆盖先加载的
```

### 3.3 Markdown 定义格式

```markdown
---
name: code-reviewer
description: Reviews code for quality and best practices
tools: [FileRead, Grep, Glob]
maxTurns: 10
background: false
color: blue
---

You are an expert code reviewer. When reviewing code:
1. Check for bugs and logic errors
2. Evaluate code style and conventions
3. Suggest improvements
...
```

### 3.4 Agent 颜色管理

```typescript
// agentColorManager.ts
const AGENT_COLORS = ['red','blue','green','yellow','purple','orange','pink','cyan']

// 每个 agentType 分配唯一颜色 (general-purpose 例外，无颜色)
// 颜色映射存储在 bootstrap state 中
getAgentColor(agentType) → Theme color key
setAgentColor(agentType, color)
```

---

## 4. Agent 生命周期

### 4.1 完整生命周期

```
模型调用 AgentTool.call()
    │
    ▼
[1. 解析] 确定 Agent 类型
    │
    ├── subagent_type → 匹配 AgentDefinition
    ├── fork → 使用父 Agent 上下文
    └── 默认 → general-purpose
    │
    ▼
[2. 准备] 构建执行环境
    │
    ├── 工具池: filterToolsForAgent() → 受限工具集
    ├── 权限: agentGetAppState() → 子 Agent 权限上下文
    ├── MCP: initializeAgentMcpServers() → 可选 MCP 连接
    ├── 消息: 构建 system prompt + user prompt
    ├── 文件缓存: clone 或新建 readFileCache
    └── 中止控制: 继承或新建 AbortController
    │
    ▼
[3. 注册] 在任务系统中注册
    │
    ├── 同步: registerAgentForeground() → 前台运行
    └── 异步: registerAsyncAgent() → 后台任务注册
    │
    ▼
[4. 执行] runAgent() → query() 推理循环
    │
    ├── 模型推理 → 工具调用 → 结果返回 → 继续推理
    ├── 进度报告: onProgress → agent_progress
    ├── 转录记录: sidechain transcript
    └── 限制: maxTurns, token budget
    │
    ▼
[5. 完成] 提取结果
    │
    ├── 同步: 提取最后 assistant 文本 → 返回 tool result
    └── 异步: completeAsyncAgent() → enqueueAgentNotification()
    │
    ▼
[6. 清理] runAgent finally 块
    │
    ├── MCP 连接断开
    ├── Session hooks 执行
    ├── 该 Agent 的 Shell 任务终止 (killShellTasksForAgent)
    ├── Monitor 任务终止
    ├── Todos 清理
    ├── 文件缓存释放
    └── 转录目录关闭
```

### 4.2 Task 状态机

```typescript
type TaskStatus = 'pending' | 'running' | 'completed' | 'failed' | 'killed'

// 状态转换:
pending → running     // 开始执行
running → completed   // 正常完成
running → failed      // 执行失败
running → killed      // 被 TaskStopTool 终止

// 终态判断:
isTerminalTaskStatus(status) = completed | failed | killed
```

### 4.3 Task ID 设计

```typescript
// 任务 ID = 类型前缀 + 8位随机字符
const TASK_ID_PREFIXES = {
  local_bash: 'b',           // b + 8 chars
  local_agent: 'a',          // a + 8 chars
  remote_agent: 'r',         // r + 8 chars
  in_process_teammate: 't',  // t + 8 chars
  local_workflow: 'w',       // w + 8 chars
  monitor_mcp: 'm',          // m + 8 chars
  dream: 'd',                // d + 8 chars
}

// 使用 36^8 ≈ 2.8 万亿组合，足以抵抗符号链接攻击
const TASK_ID_ALPHABET = '0123456789abcdefghijklmnopqrstuvwxyz'
```

---

## 5. Agent 工具隔离

### 5.1 工具过滤层级

```typescript
filterToolsForAgent({ tools, isBuiltIn, isAsync, permissionMode })

// 过滤规则:

// 1. MCP 工具始终通过
if (tool.name.startsWith('mcp__')) return true

// 2. Plan 模式下允许 ExitPlanMode
if (permissionMode === 'plan' && tool === ExitPlanMode) return true

// 3. 全局 Agent 禁止列表
ALL_AGENT_DISALLOWED_TOOLS = {
  TaskOutput,          // Agent 不应读其他 Agent 输出
  ExitPlanModeV2,      // 不应切换主会话模式
  EnterPlanMode,       // 同上
  AskUserQuestion,     // 异步 Agent 不应打断用户
  TaskStop,            // 不应停止其他任务
  Agent (非内部用户),   // 防止无限递归
  Workflow,            // 防止递归工作流
}

// 4. 异步 Agent 白名单
ASYNC_AGENT_ALLOWED_TOOLS = {
  FileRead, WebSearch, TodoWrite, Grep,
  WebFetch, Glob, Bash, PowerShell,
  FileEdit, FileWrite, NotebookEdit,
  SkillTool, ToolSearch, Worktree...
}

// 5. 协调器专用工具
COORDINATOR_MODE_ALLOWED_TOOLS = {
  Agent,            // 创建 worker
  TaskStop,         // 停止 worker
  SendMessage,      // 给 worker 发消息
  SyntheticOutput,  // 输出
}
```

### 5.2 四种角色的工具集对比

```
┌─────────────────────────────────────────────────────┐
│                    Main Thread                       │
│  所有工具 (不受 Agent 禁止列表限制)                    │
├─────────────────────────────────────────────────────┤
│                 Coordinator                          │
│  Agent + TaskStop + SendMessage + SyntheticOutput    │
├─────────────────────────────────────────────────────┤
│                Sync Sub-Agent                        │
│  大部分工具 - ALL_AGENT_DISALLOWED_TOOLS              │
│  (无 TaskOutput, AskUser, PlanMode, TaskStop)        │
├─────────────────────────────────────────────────────┤
│                Async Sub-Agent                       │
│  仅 ASYNC_AGENT_ALLOWED_TOOLS 白名单                 │
│  (FileRead, Bash, Grep, FileEdit, WebSearch...)     │
└─────────────────────────────────────────────────────┘
```

---

## 6. Agent 间通信

### 6.1 通信模型概览

```
┌────────────────────────────────────────────────────────┐
│                    通信方式矩阵                         │
├──────────────┬───────────────┬─────────────────────────┤
│  场景        │  通道          │  机制                   │
├──────────────┼───────────────┼─────────────────────────┤
│ 父→子 Agent  │ prompt 参数    │ AgentTool.call() 传入   │
│ 子→父 Agent  │ tool result   │ 同步返回 / 异步通知      │
│ 同步子完成   │ tool result   │ 直接返回最后 assistant   │
│ 异步子完成   │ task通知       │ <task-notification> XML │
│ Teammate间   │ 邮箱文件       │ teammateMailbox.ts      │
│ 广播         │ 邮箱文件       │ to: "*" → 遍历所有成员  │
│ 跨会话       │ UDS/Bridge    │ Unix Socket / HTTP      │
│ 协调器→Worker│ AgentTool     │ 创建 + SendMessage       │
│ Worker→协调器│ 完成通知       │ <task-notification>     │
└──────────────┴───────────────┴─────────────────────────┘
```

### 6.2 SendMessageTool — 核心通信工具

```typescript
// 目标寻址方式:
{
  "to": "researcher"              // Teammate 名称
  "to": "*"                       // 广播给所有 Teammate
  "to": "uds:/tmp/cc-socks/1234.sock"  // UDS 跨会话
  "to": "bridge:session_01AbCd..."     // Remote Control 跨机器
}
```

#### Teammate 通信 (邮箱机制)

```
Agent A 调用 SendMessage(to: "researcher", message: "开始任务1")
    │
    ▼
writeToMailbox("researcher", {...}, teamName)
    │
    ▼
写入文件: .claude/teams/{team}/inboxes/researcher.json
    │
    ▼
Agent "researcher" 在下一个工具轮次读取邮箱
    │
    ▼
消息作为 user role 注入到 researcher 的上下文
```

#### 结构化协议消息

```typescript
// 关闭请求/响应
{ type: "shutdown_request", request_id: "..." }
{ type: "shutdown_response", request_id: "...", approve: true }

// 计划审批请求/响应  
{ type: "plan_approval_request", request_id: "...", plan: "..." }
{ type: "plan_approval_response", request_id: "...", approve: false, feedback: "..." }

// 批准关闭 → 终止该 Agent 进程
// 拒绝计划 → 发送反馈让 Teammate 修改
```

#### 同一进程内的 Agent 通信

```typescript
// 对于 LocalAgentTask (同进程子 Agent):
// 不使用文件邮箱，而是内存队列
queuePendingMessage(agentId, message)
// 或恢复暂停的后台 Agent
resumeAgentBackground(agentId, message)
```

### 6.3 异步完成通知

```xml
<!-- 异步 Agent 完成后，通知以 XML 形式注入父对话 -->
<task-notification task_id="a8x3k2m1" status="completed">
  Agent "researcher" completed: Found 47 TODO comments across 12 files.
  Results written to todo-report.md.
</task-notification>
```

父 Agent（或协调器）看到这个通知后，可以决定下一步操作。

---

## 7. Coordinator 模式 (多 Agent 编排)

### 7.1 角色分离

```
┌─────────────────────────────────────────────────┐
│              Coordinator (协调器)                 │
│  工具: Agent + TaskStop + SendMessage            │
│  职责: 任务分解、分派、监控、结果汇总              │
│  不做: 直接执行代码、读写文件                     │
└────────┬──────────────┬──────────────┬──────────┘
         │              │              │
    ┌────▼────┐    ┌────▼────┐    ┌────▼────┐
    │ Worker1 │    │ Worker2 │    │ Worker3 │
    │ (异步)  │    │ (异步)  │    │ (异步)  │
    │ Bash    │    │ Bash    │    │ Bash    │
    │ FileEdit│    │ FileEdit│    │ FileEdit│
    │ Grep    │    │ Grep    │    │ Grep    │
    └─────────┘    └─────────┘    └─────────┘
```

### 7.2 Coordinator System Prompt 要点

```typescript
getCoordinatorSystemPrompt()
// 告诉协调器:
// - 你是任务协调器，负责将工作分解并委派给 worker
// - Worker 完成后会以 <task-notification> 通知你
// - 使用 SendMessage 继续指导 worker
// - 使用 TaskStop 终止不需要的 worker
// - 不要自己执行工具调用 (Bash/FileEdit 等)
```

### 7.3 Worker 完成流

```
Worker 完成任务
    │
    ▼
completeAsyncAgent()
    │
    ▼
enqueueAgentNotification()
    │
    ▼
<task-notification> 注入 Coordinator 对话
    │
    ▼
Coordinator 分析结果
    │
    ├── 需要更多工作 → SendMessage 给 Worker 或创建新 Worker
    ├── 所有任务完成 → 汇总结果返回用户
    └── Worker 失败 → 重试或报告错误
```

---

## 8. Swarm 团队模式

### 8.1 团队创建

```typescript
// TeamCreateTool 流程:
TeamCreateTool.call({ team_name, members })
    │
    ├── 写入团队文件: .claude/teams/{team}/team.json
    │   {
    │     name: "research-team",
    │     lead: "team-lead",
    │     members: [
    │       { name: "researcher", agentType: "researcher", status: "active" },
    │       { name: "reviewer", agentType: "code-reviewer", status: "active" }
    │     ]
    │   }
    │
    ├── 设置 AppState.teamContext
    │
    └── 注册 team-lead (当前 Agent) 为 lead
```

### 8.2 Teammate 生成策略

```
spawnTeammate(name, agentType, prompt)
    │
    ├── 后端选择:
    │   ├── in-process (同进程): 使用 runAgent + AsyncLocalStorage 隔离
    │   ├── tmux pane (分屏): 新 tmux 窗格中启动独立 claude 进程
    │   └── iTerm split: macOS 上的 iTerm 分屏
    │
    ├── 失败回退:
    │   └── pane 失败 → 自动回退到 in-process (auto 模式下)
    │
    └── 通信初始化:
        ├── in-process: 直接传入 prompt (不使用邮箱，避免重复)
        └── pane: 通过邮箱发送初始指令
```

### 8.3 团队通信拓扑

```
                    ┌─────────┐
              ┌────▶│ Teammate │◀────┐
              │     │    A     │     │
              │     └────┬─────┘     │
              │          │           │
         邮箱  │     邮箱  │      邮箱  │
              │          ▼           │
         ┌────┴─────────────────────┴────┐
         │         Team Lead             │
         │  (广播: to="*" → 遍历成员)     │
         └────┬─────────────────────┬────┘
              │                     │
         邮箱  │                邮箱  │
              ▼                     ▼
         ┌─────────┐          ┌─────────┐
         │ Teammate │          │ Teammate │
         │    B     │          │    C     │
         └─────────┘          └─────────┘

邮箱路径: .claude/teams/{team}/inboxes/{agent-name}.json
```

---

## 9. Agent 执行隔离

### 9.1 createSubagentContext — 子 Agent 上下文构建

```typescript
createSubagentContext({
  parentContext,
  isAsync,
  agentId,
  ...
})

// 隔离维度:

// 1. 状态隔离
setAppState: isAsync ? noOp : parentSetAppState  // 异步不共享状态
setAppStateForTasks: 始终指向根 store            // 任务注册/停止需要全局

// 2. 文件缓存隔离
readFileCache: fork ? cloneParentCache : newCache  // fork 继承，其他独立

// 3. 中止隔离
abortController: isAsync ? new AbortController() : parentController
// 异步 Agent 有自己的中止信号
// 同步 Agent 共享父级信号 (父级取消 → 子级也取消)

// 4. 权限隔离
permissionMode: agent.permissionMode ?? parent.mode
shouldAvoidPermissionPrompts: isAsync (后台不应弹权限确认)

// 5. 查询深度追踪
queryDepth: parent + 1
chainId: 继承
```

### 9.2 Worktree 隔离

```
EnterWorktreeTool:
    │
    ├── 创建 Git worktree: git worktree add -b session-{id} {path}
    ├── 切换 CWD: process.chdir(worktreePath)
    ├── 更新全局状态: setCwd, setOriginalCwd
    ├── 清除缓存: 系统提示词、记忆文件
    └── 保存状态: saveWorktreeState (用于 resume)

ExitWorktreeTool:
    │
    ├── 切回原始 CWD
    ├── 可选: 合并分支
    └── 清理 worktree
```

### 9.3 远程隔离 (Ant 内部)

```
isolation: 'remote' → 在云端创建隔离环境
    │
    ├── teleportToRemote(): 传送到远程会话
    ├── registerRemoteAgentTask(): 注册远程任务
    └── 通过 polling 监控远程状态
```

---

## 10. 父 Agent 如何监控子 Agent

### 10.1 同步 Agent

```
父 Agent 调用 AgentTool
    │
    ▼
runAgent() → 流式返回 Message
    │
    ├── onProgress: agent_progress 事件
    │   - 当前状态文本
    │   - Bash 进度转发
    │   - Token 用量更新
    │
    ├── BackgroundHint (2秒后):
    │   - 显示 "可以按 Ctrl+B 转为后台运行"
    │
    └── 完成: 提取最后 assistant 文本作为 tool result
```

### 10.2 异步 Agent

```
父 Agent 调用 AgentTool(run_in_background=true)
    │
    ▼
registerAsyncAgent() → 返回 { task_id, output_file }
    │
    ├── 父 Agent 继续其他工作
    │
    ├── 异步 Agent 在后台执行:
    │   └── 进度写入 outputFile
    │
    ├── 父 Agent 可以:
    │   ├── Read(output_file) → 查看进度
    │   ├── TaskStop(task_id) → 终止
    │   └── SendMessage(agent_name) → 发送指令
    │
    └── 异步 Agent 完成:
        └── <task-notification> 注入父对话
```

### 10.3 AppState.tasks — 任务状态存储

```typescript
// 所有活跃任务都在 AppState.tasks 中
AppState.tasks: Record<string, TaskStateBase> = {
  "a8x3k2m1": {
    id: "a8x3k2m1",
    type: "local_agent",
    status: "running",
    description: "TODO scanner",
    startTime: 1712345678000,
    outputFile: "/path/to/output",
    outputOffset: 1024,
    notified: false,
  }
}
```

---

## 11. Agent 系统的完整执行流 (端到端)

```
用户: "帮我审查这个 PR 并修复所有问题"
    │
    ▼
[主 Agent 推理]
  → "这个任务可以分解：一个 Agent 审查，一个 Agent 修复"
    │
    ▼
[调用 AgentTool #1]
  prompt: "审查 PR #123 的代码变更"
  subagent_type: "code-reviewer"
  run_in_background: true
    │
    ├── 解析 AgentDefinition (从 .claude/agents/ 或内建)
    ├── filterToolsForAgent → 异步白名单工具
    ├── registerAsyncAgent → task_id: "a7b2c9d1"
    ├── runAgent → query() 循环开始
    │   └── Agent 使用 Bash(gh pr diff), FileRead, Grep 审查代码
    └── 返回 { task_id: "a7b2c9d1", output_file: "..." }
    │
    ▼
[调用 AgentTool #2]
  prompt: "根据审查结果修复 src/utils.ts 中的问题"
  subagent_type: "general-purpose"
  run_in_background: true
    │
    └── 类似流程...
    │
    ▼
[主 Agent 继续工作/等待]
    │
    ├── <task-notification task_id="a7b2c9d1" status="completed">
    │   审查完成: 发现 3 个问题...
    │
    ├── <task-notification task_id="a5e8f3g2" status="completed">
    │   修复完成: 已修改 2 个文件...
    │
    ▼
[主 Agent 汇总]
  → "审查和修复都已完成。总结如下：..."
  → 返回给用户
```

---

## 12. 设计亮点

### 12.1 声明式 Agent 定义

Agent 通过 Markdown 定义，与命令/技能系统统一，支持多级覆盖（built-in → plugin → user → project → managed）。

### 12.2 统一的任务抽象

7 种任务类型（bash、agent、remote、teammate、workflow、monitor、dream）共用同一套 `Task` 接口和 `TaskStatus` 状态机。

### 12.3 灵活的通信拓扑

- 同进程: 内存队列 (`queuePendingMessage`)
- 跨进程同机: 文件邮箱 (`.claude/teams/inboxes/`)
- 跨机器: UDS Socket / Remote Control Bridge
- 结构化协议: shutdown/plan-approval 的请求/响应模式

### 12.4 渐进式隔离

```
最小隔离: 同步子 Agent (共享状态、共享 AbortController)
    ↓
中等隔离: 异步子 Agent (独立状态、独立 AbortController)
    ↓
文件隔离: Worktree (独立 Git 分支和工作目录)
    ↓
完全隔离: Remote (云端独立环境)
```

### 12.5 安全的工具降权

不同角色有严格的工具集限制，防止：
- 无限递归 (Agent 不能创建 Agent，除非内部用户)
- 权限提升 (子 Agent 不能修改权限模式)
- 用户打断 (异步 Agent 不能弹确认框)
- 递归工作流 (Agent 不能启动 Workflow)
