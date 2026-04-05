---
title: Claude Code Task 系统
aliases: [Task 系统, Task System]
series: Claude Code 架构解析
category: 工具与Agent
order: 10
tags:
  - claude-code
  - task
  - background-tasks
  - swarm
  - file-lock
date: 2026-04-04
---

# Claude Code Task 系统深度技术文档

## 1. 概述

Claude Code 中存在**两套完全独立的 Task 系统**，分别服务于不同的目的：

| 维度 | Runtime Task Framework | Swarm Task List |
|------|----------------------|-----------------|
| 定位 | 管理后台进程的生命周期 | 多 Agent 之间的工作分配协调 |
| 存储 | `AppState.tasks` 内存 + 磁盘输出文件 | `~/.claude/tasks/<listId>/` JSON 文件 |
| 核心文件 | `src/Task.ts`, `src/tasks.ts`, `src/utils/task/` | `src/utils/tasks.ts` |
| 并发控制 | `setAppState` 不可变更新 | `proper-lockfile` 文件锁 |
| 使用者 | Agent Loop、BashTool、AgentTool | Coordinator、Swarm Team |

---

## 2. Runtime Task Framework（运行时后台任务）

### 2.1 核心抽象

#### TaskType — 7 种后台任务类型

```typescript
type TaskType =
  | 'local_bash'          // 后台 Shell 命令
  | 'local_agent'         // 后台子 Agent
  | 'remote_agent'        // 远程 Agent
  | 'in_process_teammate' // 进程内 Swarm 队友
  | 'local_workflow'      // 本地 Workflow 脚本（feature-gated）
  | 'monitor_mcp'         // MCP Monitor（feature-gated）
  | 'dream'               // AutoDream 记忆整理
```

#### TaskStatus — 状态机

```
pending → running → completed
                  → failed
                  → killed
```

`isTerminalTaskStatus()` 判断任务是否已进入终态（completed / failed / killed），用于阻止向已结束的队友注入消息、触发 GC 等。

#### TaskStateBase — 基础状态字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | `string` | 唯一 ID，格式为 `<prefix><8chars>` |
| `type` | `TaskType` | 任务类型 |
| `status` | `TaskStatus` | 当前状态 |
| `description` | `string` | 人类可读描述 |
| `toolUseId` | `string?` | 关联的父工具调用 ID |
| `startTime` | `number` | 开始时间戳 |
| `endTime` | `number?` | 结束时间戳 |
| `outputFile` | `string` | 磁盘输出文件路径 |
| `outputOffset` | `number` | 增量读取偏移量 |
| `notified` | `boolean` | 是否已通知父 Agent |

#### Task ID 生成

```typescript
const TASK_ID_PREFIXES: Record<string, string> = {
  local_bash: 'b',
  local_agent: 'a',
  remote_agent: 'r',
  in_process_teammate: 't',
  local_workflow: 'w',
  monitor_mcp: 'm',
  dream: 'd',
}
```

使用 `randomBytes(8)` 生成 8 位字符（36^8 ≈ 2.8 万亿组合），前缀标识类型。

### 2.2 任务注册表

`src/tasks.ts` 维护所有任务实现的注册：

```typescript
function getAllTasks(): Task[] {
  const tasks: Task[] = [
    LocalShellTask,     // 后台 Shell
    LocalAgentTask,     // 后台 Agent
    RemoteAgentTask,    // 远程 Agent
    DreamTask,          // 记忆整理
  ]
  if (LocalWorkflowTask) tasks.push(LocalWorkflowTask)  // feature-gated
  if (MonitorMcpTask) tasks.push(MonitorMcpTask)         // feature-gated
  return tasks
}
```

每个 `Task` 实现只需提供 `name`、`type` 和 `kill()` 方法。

### 2.3 具体任务实现

#### LocalShellTask（后台 Shell 命令）

**源文件**：`src/tasks/LocalShellTask/LocalShellTask.tsx`

**关键特性**：
- **Spawn / Background 双路径**：`spawnShellTask()` 直接后台启动；`registerForeground()` 先前台运行，超时后 `backgroundExistingForegroundTask()` 切换到后台
- **Stall Watchdog**：每 5 秒轮询输出文件大小，45 秒无新输出且末行匹配交互式 Prompt 模式（如 `(y/n)`、`Continue?`）时，发送通知建议 kill 后用管道输入重跑
- **完成通知**：通过 `<task_notification>` XML 消息入队，包含 task_id、output_file、status、summary
- **批量后台化**：`backgroundAll()` 支持 Ctrl+B 一键后台化所有前台任务（Bash + Agent）

#### LocalAgentTask（后台子 Agent）

**源文件**：`src/tasks/LocalAgentTask/LocalAgentTask.tsx`

**关键特性**：
- **异步 vs 前台两种注册方式**：
  - `registerAsyncAgent()` — 直接后台，`isBackgrounded: true`
  - `registerAgentForeground()` — 先前台，通过 `backgroundSignal` Promise 等待后台化信号
- **Auto-Background**：可配置 `autoBackgroundMs`，超时自动转后台
- **Progress 追踪**：`ProgressTracker` 实时统计 tool use 次数、token 消耗、最近 5 条 Tool Activity
- **Summary 摘要**：周期性后台摘要服务调用 `updateAgentSummary()`，1-2 句话描述当前进展
- **消息队列**：`pendingMessages` 存储通过 `SendMessage` 发来的中间消息，在 tool-round 边界 drain
- **Panel 展示**：`isPanelAgentTask()` 区分 Coordinator Panel 展示的任务 vs 背景 pill 展示的任务
- **Retain 机制**：UI 持有（`retain: true`）阻止 eviction，支持 stream-append 和 disk bootstrap
- **Evict 策略**：终态后设置 `evictAfter = Date.now() + PANEL_GRACE_MS (30s)`，宽限期过后可 GC

#### InProcessTeammateTask（进程内队友）

**源文件**：`src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx`

与 `LocalAgentTask` 不同，进程内队友：
1. 运行在**同一 Node.js 进程**中，使用 `AsyncLocalStorage` 隔离
2. 有团队身份（`agentName@teamName`）
3. 支持 `shutdownRequested` 优雅关闭
4. 支持 `pendingUserMessages` 让用户在查看队友 transcript 时直接发消息
5. `appendCappedMessage()` 限制消息列表长度，防止内存膨胀

#### DreamTask（记忆整理）

**源文件**：`src/tasks/DreamTask/DreamTask.ts`

- AutoDream 系统的 UI 展现层，让本来不可见的 dream agent 出现在 footer pill 中
- 追踪 `phase`（`starting` → `updating`）、`filesTouched`、`turns`（最近 30 轮）
- Kill 时回滚 consolidation lock mtime，允许下次会话重试
- `notified: true` 立即设置（dream 不走 model-facing 通知路径）

### 2.4 Framework 核心机制

**源文件**：`src/utils/task/framework.ts`

#### 状态管理

```
registerTask(task, setAppState)     // 注册到 AppState.tasks，发射 SDK task_started
updateTaskState(id, setAppState, f) // 不可变更新，引用不变则跳过 re-render
evictTerminalTask(id, setAppState)  // 终态 + notified + 宽限期过 → 从 AppState 移除
```

**Replace-on-Resume**：`registerTask` 检测到已存在的 task（resume 场景），carry forward `retain`、`startTime`、`messages`、`diskLoaded`、`pendingMessages`，保持 Panel 排序和 transcript 连续。

#### 轮询与通知

```
pollTasks(getAppState, setAppState)         // 主轮询入口
  → generateTaskAttachments(state)          // 遍历所有任务
    → getTaskOutputDelta(id, offset)        // 增量读取磁盘输出
  → applyTaskOffsetsAndEvictions(...)       // FRESH prev.tasks 合并，防止并发覆盖
  → enqueueTaskNotification(attachment)     // XML 通知入队
```

**关键常量**：
- `POLL_INTERVAL_MS = 1000` — 标准轮询间隔
- `STOPPED_DISPLAY_MS = 3000` — killed 任务展示时间
- `PANEL_GRACE_MS = 30000` — Coordinator Panel 中终态任务的保留时间

#### 通知格式

```xml
<task_notification>
  <task_id>b3f7x2ka</task_id>
  <tool_use_id>toolu_xxx</tool_use_id>
  <output_file>/tmp/.claude/session/tasks/b3f7x2ka.output</output_file>
  <status>completed</status>
  <summary>Background command "npm test" completed (exit code 0)</summary>
</task_notification>
```

通知入队后，由 `query.ts` 的主循环作为 attachment 消费，确保主 Agent 知晓后台任务状态变化。

### 2.5 磁盘输出系统

**源文件**：`src/utils/task/diskOutput.ts`

#### DiskTaskOutput 类

每个任务的输出写入独立磁盘文件，核心是 `DiskTaskOutput` 类：

- **异步写入队列**：`append(content)` 立即入队，单个 `#drain()` 循环处理
- **GC 优化**：`splice(0, len)` in-place 清空数组，单次 `Buffer.allocUnsafe` 拼接后写入，避免 `.then()` 链闭包持有数据导致内存膨胀
- **磁盘上限**：`MAX_TASK_OUTPUT_BYTES = 5GB`，超限后追加截断标记
- **安全**：`O_NOFOLLOW` 防止沙箱内 symlink 攻击；`O_EXCL` 确保创建新文件

#### 路径隔离

```typescript
function getTaskOutputDir(): string {
  return join(getProjectTempDir(), getSessionId(), 'tasks')
}
```

Session ID 在首次调用时捕获并缓存（`/clear` 不影响已有任务的输出文件路径）。

#### Agent 输出 Symlink

`initTaskOutputAsSymlink(agentId, transcriptPath)` — Agent 类任务的输出文件实际是指向 JSONL transcript 的符号链接，避免数据复制。

### 2.6 Stop / Kill 流程

**源文件**：`src/tasks/stopTask.ts`

```
TaskStopTool / SDK stop_task
  → stopTask(taskId, context)
    → getTaskByType(task.type)
    → taskImpl.kill(taskId, setAppState)
    → (Shell) 标记 notified，抑制 exit-code-137 通知
    → (Shell) emitTaskTerminatedSdk() 补偿 SDK 事件
```

**错误类型**：`StopTaskError` 区分 `not_found`、`not_running`、`unsupported_type`。

**批量 Kill**：
- `killAllRunningAgentTasks()` — ESC 取消 Coordinator 模式所有子 Agent
- `markAgentsNotified()` — 批量 kill 时抑制逐个通知，改为发送一条聚合消息

### 2.7 前台 / 后台切换

Runtime Task 支持前台 → 后台的动态切换：

```
registerForeground()          // 注册前台任务 (isBackgrounded: false)
    ↓ (运行一段时间)
backgroundAgentTask()         // 用户触发或 auto-background 定时器
    ↓
isBackgrounded: true          // 切换到后台，Agent Loop 继续运行
    ↓
完成 → enqueueNotification   // 通过 <task_notification> 通知主 Agent
```

`hasForegroundTasks()` 判断是否有可后台化的任务（Ctrl+B 行为分支：有前台任务 → 后台化；无前台任务 → 后台化会话）。

### 2.8 与 Agent Loop 的集成

`src/query.ts` 中的主循环消费任务通知：

- 主线程 drain `agentId === undefined` 的命令
- 子 Agent 只 drain `mode === 'task-notification'` 且 `agentId === currentAgentId` 的命令
- `task-notification` 在 attachment 转换后立即移除

### 2.9 SDK 集成

- `registerTask` 发射 `task_started` SDK 事件
- `stopTask` 对 Shell 类任务发射 `task_terminated` SDK 事件
- `updateAgentSummary` 在 SDK 开启 `agentProgressSummaries` 选项时发射 `task_progress` 事件

---

## 3. Swarm Task List（多 Agent 协作任务列表）

### 3.1 核心抽象

**源文件**：`src/utils/tasks.ts`

与 Runtime Task 完全无关的独立系统，是一个**文件系统上的 todo list**。

#### Task Schema

```typescript
const TaskSchema = z.object({
  id: z.string(),                                    // 自增整数字符串
  subject: z.string(),                               // 任务标题
  description: z.string(),                           // 详细描述
  activeForm: z.string().optional(),                  // 进行时描述（"Running tests"）
  owner: z.string().optional(),                       // 认领者 agent ID
  status: z.enum(['pending', 'in_progress', 'completed']),
  blocks: z.array(z.string()),                        // 本任务阻塞哪些任务
  blockedBy: z.array(z.string()),                     // 本任务被哪些任务阻塞
  metadata: z.record(z.string(), z.unknown()).optional(),
})
```

#### 存储结构

```
~/.claude/tasks/<taskListId>/
  ├── .lock                    # 列表级文件锁
  ├── .highwatermark           # 已分配的最大 ID（防止 reset 后 ID 重用）
  ├── 1.json                   # 任务 1
  ├── 2.json                   # 任务 2
  └── ...
```

### 3.2 Task List ID 解析

优先级从高到低：
1. `CLAUDE_CODE_TASK_LIST_ID` 环境变量
2. 进程内队友（`getTeammateContext().teamName`）
3. `CLAUDE_CODE_TEAM_NAME`（进程级队友环境变量）
4. `leaderTeamName`（Leader 创建 Team 时设置）
5. `getSessionId()`（独立会话回退值）

### 3.3 并发控制

10+ 并发 Agent 的 Swarm 场景下通过 `proper-lockfile` 保证一致性：

```typescript
const LOCK_OPTIONS = {
  retries: { retries: 30, minTimeout: 5, maxTimeout: 100 },
  // ~2.6s 总等待预算，足够 10-way race
}
```

- **列表级锁**：`createTask()`、`resetTaskList()` 使用列表级 `.lock` 文件
- **任务级锁**：`updateTask()`、`claimTask()` 使用单个任务文件作为锁目标
- **Busy Check 升级**：`claimTask({ checkAgentBusy: true })` 升级到列表级锁，原子检查 Agent 是否已有未完成任务

### 3.4 核心操作

| 操作 | 函数 | 说明 |
|------|------|------|
| 创建 | `createTask(listId, data)` | 加锁 → 读最大 ID → 写新 JSON |
| 更新 | `updateTask(listId, id, updates)` | 加锁 → 读 → 合并 → 写 |
| 认领 | `claimTask(listId, id, agentId, opts)` | 检查阻塞 → 检查 busy → 设置 owner |
| 删除 | `deleteTask(listId, id)` | 更新 high water mark → 删 JSON → 清引用 |
| 列表 | `listTasks(listId)` | readdir → 逐个 parse |
| 阻塞 | `blockTask(listId, from, to)` | 双向写入 blocks/blockedBy |
| 重置 | `resetTaskList(listId)` | 保存 high water mark → 删除所有 JSON |

### 3.5 Agent 状态查询

```typescript
async function getAgentStatuses(teamName: string): Promise<AgentStatus[] | null>
```

读取 Team 配置 → 读取所有任务 → 按 owner 分组 → 输出每个 Agent 的 `idle | busy` 状态和 `currentTasks` 列表。

### 3.6 队友退出处理

```typescript
async function unassignTeammateTasks(
  teamName, teammateId, teammateName, reason
): Promise<UnassignTasksResult>
```

当 Teammate 被 kill 或优雅关闭时：
1. 找到该 Teammate 所有未完成任务
2. 清除 `owner`，状态重置为 `pending`
3. 生成通知消息，建议 Leader 使用 `TaskList` 查看并用 `TaskUpdate` 重新分配

### 3.7 事件通知

```typescript
const tasksUpdated = createSignal()
export const onTasksUpdated = tasksUpdated.subscribe
```

本地 Signal 机制，任务变更后立即通知 UI 刷新。`leaderTeamName` 变更也视为 "tasks updated"（Task List ID 变了，看到的是不同目录）。

---

## 4. TaskOutputTool 和 TaskStopTool

### 4.1 TaskOutputTool

**名称**：`TaskOutput`

LLM 用来读取后台任务输出的工具：

- **参数**：`task_id`、`block`（默认 true）、`timeout`（默认 30s）
- **阻塞模式**：轮询 `getAppState().tasks[taskId]` 每 100ms 直到终态或超时
- **输出来源**：
  - `local_bash`：优先使用 shell 的 `taskOutput`，fallback 到 `getTaskOutput()`
  - `local_agent`：优先用内存 `extractTextContent(result.content)`，因为磁盘文件是指向 JSONL 的 symlink
- **输出格式**：XML tags（`<retrieval_status>`、`<task_id>`、`<status>`、`<output>`）
- **Deprecated**：Prompt 提示优先使用 Read 工具直接读 `outputFile`

### 4.2 TaskStopTool

**名称**：`TaskStop`（别名 `KillShell`）

LLM 用来终止后台任务的工具：

- **验证**：任务必须存在且 `status === 'running'`
- **执行**：`stopTask()` → `taskImpl.kill()`
- **特性**：`shouldDefer: true`、`isConcurrencySafe: true`

---

## 5. 两套系统的关系

```
┌─────────────────────────────────────────────────────────┐
│                      Swarm Task List                     │
│  "什么工作要做" — 文件系统 JSON，多 Agent 共享           │
│  createTask → claimTask → updateTask → completed         │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐                   │
│  │ Task #1 │ │ Task #2 │ │ Task #3 │                   │
│  │ owner:A │ │ owner:B │ │ pending │                   │
│  └────┬────┘ └────┬────┘ └─────────┘                   │
│       │            │                                     │
└───────┼────────────┼─────────────────────────────────────┘
        │            │
        ▼            ▼
┌─────────────────────────────────────────────────────────┐
│                  Runtime Task Framework                   │
│  "进程怎么跑" — AppState 内存，单进程管理               │
│  registerTask → running → completed/failed/killed        │
│  ┌──────────────┐ ┌──────────────┐                      │
│  │ local_agent  │ │ local_bash   │                      │
│  │ Agent A 进程 │ │ npm test 进程│                      │
│  └──────────────┘ └──────────────┘                      │
└─────────────────────────────────────────────────────────┘
```

- Swarm Task List 是**规划层**：决定谁做什么
- Runtime Task Framework 是**执行层**：管理进程生命周期
- 一个 Swarm Task 被 Agent 认领后，Agent 执行时会创建对应的 Runtime Task（如 local_bash、local_agent）
- 两套系统之间没有直接引用关系，通过 Agent 的行为间接关联

---

## 6. 关键源文件索引

| 文件 | 说明 |
|------|------|
| `src/Task.ts` | 核心类型定义（TaskType、TaskStatus、TaskStateBase、ID 生成） |
| `src/tasks.ts` | 任务实现注册表 |
| `src/tasks/types.ts` | TaskState 联合类型、`isBackgroundTask` 判定 |
| `src/tasks/stopTask.ts` | 通用 stop 编排 |
| `src/tasks/LocalShellTask/LocalShellTask.tsx` | Shell 任务全生命周期 |
| `src/tasks/LocalShellTask/guards.ts` | Shell 任务类型守卫 |
| `src/tasks/LocalShellTask/killShellTasks.ts` | Shell 任务 kill 实现 |
| `src/tasks/LocalAgentTask/LocalAgentTask.tsx` | Agent 任务全生命周期 + Progress 追踪 |
| `src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx` | 进程内队友任务 |
| `src/tasks/InProcessTeammateTask/types.ts` | 队友任务状态类型 |
| `src/tasks/DreamTask/DreamTask.ts` | Dream 记忆整理任务 |
| `src/tasks/RemoteAgentTask/RemoteAgentTask.tsx` | 远程 Agent 任务 |
| `src/tasks/LocalMainSessionTask.ts` | 主会话任务标识 |
| `src/tasks/pillLabel.ts` | 后台任务 pill 标签 |
| `src/utils/task/framework.ts` | Framework 核心（注册/更新/轮询/GC/通知） |
| `src/utils/task/diskOutput.ts` | 磁盘输出系统（DiskTaskOutput 类） |
| `src/utils/task/sdkProgress.ts` | SDK progress 事件发射 |
| `src/utils/tasks.ts` | Swarm Task List 全部逻辑 |
| `src/tools/TaskOutputTool/` | LLM 读取任务输出的工具 |
| `src/tools/TaskStopTool/` | LLM 停止任务的工具 |

---

## 7. 设计亮点

1. **不可变状态 + 引用相等跳过**：`updateTaskState` 中 updater 返回同一引用 → 跳过 spread → 避免无谓 re-render

2. **TOCTOU 安全**：`applyTaskOffsetsAndEvictions` 在 async await 后重新从 FRESH `prev.tasks` 读取，不用 stale snapshot

3. **Symlink 防攻击**：`O_NOFOLLOW` 防止沙箱内通过 symlink 劫持输出文件写入任意位置

4. **GC 友好的写入队列**：`DiskTaskOutput.#queueToBuffers()` 单独方法，确保 GC 尽快回收 queue 数组，避免 `.then()` 闭包链持有内存

5. **原子去重通知**：`enqueueAgentNotification` / `enqueueShellNotification` 通过 `updateTaskState` 原子检查并设置 `notified` flag，防止重复通知

6. **文件锁 + High Water Mark**：Swarm Task List 使用 high water mark 文件追踪历史最大 ID，即使 `resetTaskList` 删除所有任务也不会重用旧 ID
