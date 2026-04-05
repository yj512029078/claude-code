---
title: Claude Code KAIROS 主动式助手系统
aliases: [KAIROS 系统, KAIROS System, 主动式助手]
series: Claude Code 架构解析
category: 特色功能
order: 20
tags:
  - claude-code
  - kairos
  - proactive
  - always-on
  - github-webhook
date: 2026-04-04
---

# Claude Code KAIROS 主动式助手系统 — 深度技术文档

## 目录

1. [系统概述](#1-系统概述)
   - 1.1 [定位与设计理念](#11-定位与设计理念)
   - 1.2 [Feature Flag 体系](#12-feature-flag-体系)
   - 1.3 [与 PROACTIVE 模式的关系](#13-与-proactive-模式的关系)
2. [启动与激活流程](#2-启动与激活流程)
   - 2.1 [编译时门控](#21-编译时门控)
   - 2.2 [运行时激活链](#22-运行时激活链)
   - 2.3 [CLI 入口](#23-cli-入口)
   - 2.4 [Assistant 子命令 — 远程查看器](#24-assistant-子命令--远程查看器)
3. [心跳驱动循环 (Tick Loop)](#3-心跳驱动循环-tick-loop)
   - 3.1 [Tick 消息格式](#31-tick-消息格式)
   - 3.2 [REPL 模式 Tick 调度](#32-repl-模式-tick-调度)
   - 3.3 [Headless / SDK 模式 Tick 调度](#33-headless--sdk-模式-tick-调度)
   - 3.4 [Tick 的 Pause / Resume / Block 状态机](#34-tick-的-pause--resume--block-状态机)
4. [系统提示词工程](#4-系统提示词工程)
   - 4.1 [Proactive Section — 自主工作指令](#41-proactive-section--自主工作指令)
   - 4.2 [Brief Section — 通信协议指令](#42-brief-section--通信协议指令)
   - 4.3 [Assistant Addendum — KAIROS 专属补丁](#43-assistant-addendum--kairos-专属补丁)
   - 4.4 [Compact 恢复指令](#44-compact-恢复指令)
5. [SleepTool — 节拍控制器](#5-sleeptool--节拍控制器)
   - 5.1 [工具设计](#51-工具设计)
   - 5.2 [节流参数](#52-节流参数)
   - 5.3 [Prompt Cache 与成本权衡](#53-prompt-cache-与成本权衡)
6. [Cron 定时任务系统](#6-cron-定时任务系统)
   - 6.1 [任务模型](#61-任务模型)
   - 6.2 [调度器架构](#62-调度器架构)
   - 6.3 [/loop Skill](#63-loop-skill)
   - 6.4 [锁与多会话协调](#64-锁与多会话协调)
   - 6.5 [Jitter 与负载均衡](#65-jitter-与负载均衡)
7. [通信层 — SendUserMessage (Brief)](#7-通信层--sendusermessage-brief)
   - 7.1 [工具接口](#71-工具接口)
   - 7.2 [启用门控链](#72-启用门控链)
   - 7.3 [主动 vs 被动消息路由](#73-主动-vs-被动消息路由)
8. [感知层](#8-感知层)
   - 8.1 [终端焦点检测](#81-终端焦点检测)
   - 8.2 [GitHub Webhook 订阅](#82-github-webhook-订阅)
   - 8.3 [Channel 消息系统](#83-channel-消息系统)
   - 8.4 [推送通知](#84-推送通知)
9. [记忆管理 — 日志式追加 + 夜间蒸馏](#9-记忆管理--日志式追加--夜间蒸馏)
   - 9.1 [日志文件布局](#91-日志文件布局)
   - 9.2 [/dream — 夜间记忆整合](#92-dream--夜间记忆整合)
   - 9.3 [与标准 Auto Memory 的区别](#93-与标准-auto-memory-的区别)
10. [Agent 行为差异](#10-agent-行为差异)
    - 10.1 [强制异步执行](#101-强制异步执行)
    - 10.2 [Bash/PowerShell 后台化](#102-bashpowershell-后台化)
    - 10.3 [团队预初始化](#103-团队预初始化)
11. [防护与安全机制](#11-防护与安全机制)
    - 11.1 [用户中断 → 暂停](#111-用户中断--暂停)
    - 11.2 [API 错误阻断](#112-api-错误阻断)
    - 11.3 [Context 压缩恢复](#113-context-压缩恢复)
    - 11.4 [远程 Kill Switch](#114-远程-kill-switch)
    - 11.5 [目录信任检查](#115-目录信任检查)
12. [完整生命周期](#12-完整生命周期)
13. [架构总结 — 守护进程类比](#13-架构总结--守护进程类比)

---

## 1. 系统概述

### 1.1 定位与设计理念

KAIROS 是 Claude Code 内部的**主动式助手模式 (Assistant Mode)**，将 Claude 从一个"用户问→AI答"的响应式 CLI 工具，转变为一个**始终在线、自主决策的开发守护进程**。

核心设计理念：

- **不等待指令**：通过 tick 心跳驱动自主循环，LLM 持续评估当前上下文并决定下一步行动
- **感知用户状态**：检测终端焦点、订阅外部事件，动态调整自主程度
- **异步通信**：通过 `SendUserMessage` 通道与用户异步交互，用户无需盯屏
- **持久记忆**：日志式追加 + 夜间蒸馏，跨会话保持上下文

### 1.2 Feature Flag 体系

KAIROS 通过 Bun bundler 的 `feature()` 宏实现**编译时条件编译**，确保外部构建不包含任何 KAIROS 代码：

| Flag | 功能 | 说明 |
|------|------|------|
| `KAIROS` | 主体模式 | 主动助手核心逻辑 |
| `KAIROS_BRIEF` | 简报工具 | `SendUserMessage` 独立发布通道 |
| `KAIROS_DREAM` | 记忆整合 | `/dream` skill 独立发布通道 |
| `KAIROS_GITHUB_WEBHOOKS` | GitHub 事件 | PR/Issue 事件订阅 |
| `KAIROS_PUSH_NOTIFICATION` | 推送通知 | 系统级推送 |
| `KAIROS_CHANNELS` | Channel 消息 | MCP 通道推送 |

```typescript
// src/main.tsx — 编译时导入门控
const assistantModule = feature('KAIROS')
  ? require('./assistant/index.js') : null
const kairosGate = feature('KAIROS')
  ? require('./assistant/gate.js') : null
```

当 `feature('KAIROS')` 为 `false` 时，Bun bundler 的 dead-code elimination 会将整个 KAIROS 子系统从产物中移除。

### 1.3 与 PROACTIVE 模式的关系

代码中大量出现 `feature('PROACTIVE') || feature('KAIROS')` 模式：

```typescript
// src/utils/systemPrompt.ts
const proactiveModule =
  feature('PROACTIVE') || feature('KAIROS')
    ? require('../proactive/index.js') : null
```

- **PROACTIVE** 是较早的通用主动模式，通过 `--proactive` 或 `CLAUDE_CODE_PROACTIVE=1` 激活
- **KAIROS** 是 PROACTIVE 的超集，在其基础上增加了 Assistant 模块、Brief 通信、Cron 调度、GitHub Webhook 等

两者共享 `proactive/` 模块中的 tick 调度、pause/resume 等基础设施。

---

## 2. 启动与激活流程

### 2.1 编译时门控

KAIROS 相关代码通过 `feature()` 宏进行编译时分支：

```typescript
// 编译时为 false → 整个 require + 所有引用被 tree-shaking 消除
if (feature('KAIROS')) { ... }
```

外部发布的 npm 包中，所有 KAIROS 分支均被消除，不会泄露任何 assistant 逻辑。

### 2.2 运行时激活链

```
┌─────────────────────────────────────────────────────────┐
│  feature('KAIROS') === true (编译时)                     │
│  ↓                                                      │
│  isAssistantMode() → 检查 .claude/agents/assistant.md   │
│  ↓                                                      │
│  checkHasTrustDialogAccepted() → 目录已信任？            │
│  ↓                                                      │
│  kairosGate.isKairosEnabled() → GrowthBook gate         │
│    ├─ 磁盘缓存 hit → 立即返回                           │
│    └─ 磁盘缓存 miss → 初始化 GrowthBook, 远程拉取 (≤5s)│
│  ↓                                                      │
│  setKairosActive(true)                                  │
│  opts.brief = true (强制开启 Brief)                      │
│  initializeAssistantTeam() (预建团队)                    │
└─────────────────────────────────────────────────────────┘
```

源码对应 `src/main.tsx:1048-1086`：

```typescript
let kairosEnabled = false;

if (feature('KAIROS') && assistantModule?.isAssistantMode() &&
    !(options as { agentId?: unknown }).agentId && kairosGate) {
  if (!checkHasTrustDialogAccepted()) {
    console.warn(chalk.yellow(
      'Assistant mode disabled: directory is not trusted.'
    ));
  } else {
    kairosEnabled = assistantModule.isAssistantForced()
      || (await kairosGate.isKairosEnabled());
    if (kairosEnabled) {
      opts.brief = true;
      setKairosActive(true);
      assistantTeamContext =
        await assistantModule.initializeAssistantTeam();
    }
  }
}
```

关键细节：
- **Spawned Teammate 不重复初始化**：通过 `!(options as { agentId }).agentId` 检查，避免被 spawn 出的队友重复初始化
- **`--assistant` 强制模式**：Agent SDK daemon 场景下可跳过 GrowthBook gate

### 2.3 CLI 入口

```typescript
// src/main.tsx:3832-3843
if (feature('PROACTIVE') || feature('KAIROS')) {
  program.addOption(new Option('--proactive',
    'Start in proactive autonomous mode'));
}
if (feature('KAIROS')) {
  program.addOption(new Option('--assistant',
    'Force assistant mode (Agent SDK daemon use)').hideHelp());
}
if (feature('KAIROS') || feature('KAIROS_BRIEF')) {
  program.addOption(new Option('--brief',
    'Enable SendUserMessage tool for agent-to-user communication'));
}
```

### 2.4 Assistant 子命令 — 远程查看器

```bash
claude assistant [sessionId]
```

这个子命令将 REPL 作为**纯查看器客户端**连接到远程助手会话：
- 智能循环在远程运行
- 本地进程流式显示事件并发送消息
- 历史记录通过 `useAssistantHistory` 懒加载
- 如果不指定 sessionId，会自动发现可用会话

---

## 3. 心跳驱动循环 (Tick Loop)

Tick 是 KAIROS 主动循环的核心驱动力。它将 LLM 的交互从"请求-响应"模式变为"持续运转"模式。

### 3.1 Tick 消息格式

```typescript
// src/constants/xml.ts
export const TICK_TAG = 'tick'

// 生成的 tick 内容示例
`<tick>14:30:05</tick>`
```

Tick 消息以 XML 标签包裹当前本地时间，作为一条 `user` 角色消息注入对话流。它携带 `isMeta: true` 标记，使其：
- 不显示在 UI 消息列表中
- 不在转录日志中显示为用户输入
- 不可被用户编辑/撤回

### 3.2 REPL 模式 Tick 调度

在交互式 REPL 中，tick 通过 `useProactive` hook 管理：

```typescript
// src/screens/REPL.tsx:4079-4095
useProactive?.({
  isLoading: isLoading || initialMessage !== null,
  queuedCommandsLength: queuedCommands.length,
  hasActiveLocalJsxUI: isShowingLocalJSXCommand,
  isInPlanMode: toolPermissionContext.mode === 'plan',
  onSubmitTick: (prompt: string) => handleIncomingPrompt(prompt, {
    isMeta: true
  }),
  onQueueTick: (prompt: string) => enqueue({
    mode: 'prompt',
    value: prompt,
    isMeta: true
  })
});
```

Tick 抑制条件：
- `isLoading` 为 true（正在处理中）
- `initialMessage` 待处理（防止与启动消息竞争）
- 显示本地 JSX 命令（用户正在交互）
- 处于 Plan 模式

### 3.3 Headless / SDK 模式 Tick 调度

在非交互模式（`-p` 或 SDK）中，tick 通过 `scheduleProactiveTick` 函数管理：

```typescript
// src/cli/print.ts:1831-1856
const scheduleProactiveTick =
  feature('PROACTIVE') || feature('KAIROS')
    ? () => {
        setTimeout(() => {
          if (
            !proactiveModule?.isProactiveActive() ||
            proactiveModule.isProactivePaused() ||
            inputClosed
          ) {
            return
          }
          const tickContent =
            `<${TICK_TAG}>${new Date().toLocaleTimeString()}</${TICK_TAG}>`
          enqueue({
            mode: 'prompt' as const,
            value: tickContent,
            uuid: randomUUID(),
            priority: 'later',
            isMeta: true,
          })
          void run()
        }, 0)
      }
    : undefined
```

**关键设计**：`setTimeout(0)` 将 tick 推迟到当前事件循环尾部，确保 pending 的 stdin 消息（用户中断、新输入）优先处理。

### 3.4 Tick 的 Pause / Resume / Block 状态机

```
                ┌──────────┐
  用户按 Esc ──→│  Paused   │←── API Error
                └────┬─────┘
                     │
  用户输入 ──→ resumeProactive()
                     │
                ┌────▼─────┐
                │  Active   │──→ tick 正常发送
                └────┬─────┘
                     │
  Context 满 ──→ setContextBlocked(true)
                     │
                ┌────▼──────────┐
                │ContextBlocked │──→ tick 暂停
                └────┬──────────┘
                     │
  Compact 完成 ──→ setContextBlocked(false)
                     │
                ┌────▼─────┐
                │  Active   │
                └──────────┘
```

状态转换的源码位置：

```typescript
// 用户按 Esc → 暂停
// src/screens/REPL.tsx:2113-2117
if (feature('PROACTIVE') || feature('KAIROS')) {
  proactiveModule?.pauseProactive();
}

// 用户提交输入 → 恢复
// src/screens/REPL.tsx:3153-3156
if (feature('PROACTIVE') || feature('KAIROS')) {
  proactiveModule?.resumeProactive();
}

// API 错误 → 阻断
// src/screens/REPL.tsx:2631-2639
if (newMessage.isApiErrorMessage) {
  proactiveModule?.setContextBlocked(true);
} else if (newMessage.type === 'assistant') {
  proactiveModule?.setContextBlocked(false);
}

// Compact 完成 → 解除阻断
// src/screens/REPL.tsx:2604-2607
if (feature('PROACTIVE') || feature('KAIROS')) {
  proactiveModule?.setContextBlocked(false);
}
```

---

## 4. 系统提示词工程

KAIROS 模式通过多层系统提示词注入来引导 Claude 的行为。

### 4.1 Proactive Section — 自主工作指令

`src/constants/prompts.ts` 中的 `getProactiveSection()` 生成核心自主工作指令：

```
# Autonomous work

You are running autonomously. You will receive `<tick>` prompts
that keep you alive between turns — just treat them as
"you're awake, what now?"
```

**关键行为规则**：

| 规则 | 内容 |
|------|------|
| **Pacing** | 用 Sleep 控制等待时长；长等长睡，快速迭代短睡 |
| **空闲必须 Sleep** | 无事可做时**必须**调用 Sleep，禁止输出 "still waiting" 之类的废话 |
| **首次唤醒** | 简短问候，询问用户想做什么，不要自行探索 |
| **后续唤醒** | 寻找有用的工作；不要重复提问；不要描述要做什么——直接做 |
| **终端聚焦** | Focused → 协作式、多请示；Unfocused → 全自主、大胆行动 |
| **行动偏向** | 优先行动而非确认：读文件、跑测试、搜代码——无需许可 |
| **简洁输出** | 只输出决策点、里程碑、错误——不要逐步解说 |

### 4.2 Brief Section — 通信协议指令

当 Brief 工具可用时，注入通信协议指令：

```
## Talking to the user

SendUserMessage is where your replies go. Text outside it is visible
if the user expands the detail view, but most won't — assume unread.

So: every time the user says something, the reply they actually read
comes through SendUserMessage. Even for "hi". Even for "thanks".

If you can answer right away, send the answer. If you need to go
look — ack first in one line ("On it — checking the test output"),
then work, then send the result. Without the ack they're staring
at a spinner.

For longer work: ack → work → result. Between those, send a
checkpoint when something useful happened.
```

**设计要点**：
- 所有用户可见内容必须通过 `SendUserMessage`，工具调用外的文本被视为"详情视图"
- 避免"失败模式"：真正的回答在纯文本中，Brief 只说了 "done!"
- 长任务三段式：ack → 工作 → 结果；中间按里程碑发 checkpoint

### 4.3 Assistant Addendum — KAIROS 专属补丁

```typescript
// src/main.tsx:2206-2208
if (feature('KAIROS') && kairosEnabled && assistantModule) {
  const assistantAddendum =
    assistantModule.getAssistantSystemPromptAddendum();
  appendSystemPrompt = appendSystemPrompt
    ? `${appendSystemPrompt}\n\n${assistantAddendum}`
    : assistantAddendum;
}
```

Assistant 模块追加额外的系统提示词，包含 KAIROS 专属的行为指导（如预建团队使用说明等）。

### 4.4 Compact 恢复指令

上下文压缩后注入的恢复提示：

```typescript
// src/services/compact/prompt.ts:362-367
if (proactiveModule?.isProactiveActive()) {
  continuation += `
You are running in autonomous/proactive mode. This is NOT a first
wake-up — you were already working autonomously before compaction.
Continue your work loop: pick up where you left off based on the
summary above. Do not greet the user or ask what to work on.`
}
```

防止 Claude 在 compact 后误以为是新会话而重新问候用户。

---

## 5. SleepTool — 节拍控制器

### 5.1 工具设计

```typescript
// src/tools/SleepTool/prompt.ts
export const SLEEP_TOOL_PROMPT = `Wait for a specified duration.
The user can interrupt the sleep at any time.

Use this when the user tells you to sleep or rest, when you have
nothing to do, or when you're waiting for something.

You may receive <tick> prompts — these are periodic check-ins.
Look for useful work to do before sleeping.

You can call this concurrently with other tools — it won't
interfere with them.

Prefer this over \`Bash(sleep ...)\` — it doesn't hold a shell process.

Each wake-up costs an API call, but the prompt cache expires after
5 minutes of inactivity — balance accordingly.`
```

**关键特性**：
- 不占用 shell 进程（与 `Bash(sleep)` 的区别）
- 可与其他工具并发调用
- 用户随时可中断

### 5.2 节流参数

```typescript
// src/utils/settings/types.ts:841-863
minSleepDurationMs: z.number().nonnegative().int().optional()
  .describe('Minimum duration in milliseconds that the Sleep tool
    must sleep for. Useful for throttling proactive tick frequency.'),

maxSleepDurationMs: z.number().int().min(-1).optional()
  .describe('Maximum duration in milliseconds that the Sleep tool
    can sleep for. Set to -1 for indefinite sleep (waits for user
    input). Useful for limiting idle time in remote/managed
    environments.'),
```

- `minSleepDurationMs`：防止 Claude 过于频繁唤醒（节流 API 成本）
- `maxSleepDurationMs`：限制最大空闲时间；`-1` 表示无限等待直到用户输入

### 5.3 Prompt Cache 与成本权衡

系统提示词明确告知 Claude 成本模型：

> "Each wake-up costs an API call, but the prompt cache expires after 5 minutes of inactivity — balance accordingly."

Claude 需要在两个目标间平衡：
- 频繁唤醒 → 更快响应，但每次唤醒消耗一次 API 调用
- 间隔 > 5 分钟 → prompt cache 过期，下次调用需要重新预填充缓存

最优策略是保持 < 5 分钟的 sleep 间隔以维持 cache 热度。

---

## 6. Cron 定时任务系统

### 6.1 任务模型

```typescript
// src/utils/cronTasks.ts
type CronTask = {
  id: string
  cron: string            // 5 字段 cron 表达式（本地时间）
  prompt: string          // 触发时入队的 prompt
  createdAt: number       // 创建时间 (epoch ms)
  lastFiredAt?: number    // 最近触发时间
  recurring?: boolean     // true=循环，false=一次性
  permanent?: boolean     // true=豁免自动过期（系统任务专用）
  durable?: boolean       // false=仅内存（会话级），undefined=持久化到文件
  agentId?: string        // 运行时：由哪个 teammate 创建
}
```

**两类任务**：
- **一次性 (One-shot)**：触发后自动删除。适用于 "提醒我下午3点..."
- **循环 (Recurring)**：按 cron 表达式反复触发，默认 10 天后自动过期。`permanent: true` 的任务（如 assistant 内置的晨检、记忆整合）不过期

**两类存储**：
- **持久化 (Durable)**：写入 `.claude/scheduled_tasks.json`，跨会话存活
- **会话级 (Session)**：仅存在于 `bootstrap/state.ts` 内存中，进程退出即消失

### 6.2 调度器架构

```
┌─────────────────────────────────────────────────────┐
│                  CronScheduler                       │
│                                                      │
│  ┌──────────────┐     ┌──────────────────────────┐  │
│  │ enablePoll   │     │ checkTimer (1s interval)  │  │
│  │ 等待首个任务 │────→│ check() 每秒执行          │  │
│  └──────────────┘     └──────────┬───────────────┘  │
│                                  │                   │
│        ┌─────────────────────────┤                   │
│        │                         │                   │
│  ┌─────▼──────┐          ┌──────▼───────┐           │
│  │ File Tasks  │          │Session Tasks │           │
│  │(JSON 文件)  │          │(内存)         │           │
│  │ 需 Lock     │          │ 无 Lock      │           │
│  └─────┬──────┘          └──────┬───────┘           │
│        │                         │                   │
│  ┌─────▼─────────────────────────▼────────┐         │
│  │  process(task)                          │         │
│  │  ├ 检查 nextFireAt                     │         │
│  │  ├ now >= next → 触发                  │         │
│  │  │   ├ onFireTask(task) 或 onFire(prompt)        │
│  │  │   ├ recurring → 重新计算 nextFireAt │         │
│  │  │   └ one-shot → 删除任务             │         │
│  │  └ now < next → 跳过                   │         │
│  └────────────────────────────────────────┘         │
│                                                      │
│  ┌──────────────────────────────────────┐           │
│  │ chokidar 文件监听                     │           │
│  │ scheduled_tasks.json 变更 → reload   │           │
│  └──────────────────────────────────────┘           │
└─────────────────────────────────────────────────────┘
```

调度器由 `createCronScheduler()` 创建，核心参数：

```typescript
type CronSchedulerOptions = {
  onFire: (prompt: string) => void       // 触发回调
  onFireTask?: (task: CronTask) => void  // 触发回调（带完整任务）
  onMissed?: (tasks: CronTask[]) => void // 错过的任务回调
  isLoading: () => boolean               // 当前是否在处理
  assistantMode?: boolean                // KAIROS 模式（绕过 isLoading）
  dir?: string                           // 任务文件目录（daemon 模式）
  lockIdentity?: string                  // 锁身份标识
  getJitterConfig?: () => CronJitterConfig // jitter 配置
  isKilled?: () => boolean               // kill switch
  filter?: (t: CronTask) => boolean      // 任务过滤器
}
```

### 6.3 /loop Skill

`/loop` 是面向用户的便捷封装，将自然语言间隔转为 cron 表达式：

```
/loop 5m /babysit-prs        → */5 * * * * (每5分钟)
/loop check the deploy        → */10 * * * * (默认10分钟)
/loop 30m check the deploy    → */30 * * * *
/loop 2h run tests           → 0 */2 * * *
/loop check deploy every 20m → */20 * * * *
```

**行为**：
1. 解析间隔和 prompt
2. 调用 `CronCreateTool` 创建循环任务
3. **立即执行一次**（不等首次 cron 触发）
4. 告知用户 cron 表达式、过期时间、取消方法

### 6.4 锁与多会话协调

当多个 Claude 会话共享同一 cwd 时，调度器使用文件锁防止重复触发：

```typescript
// 只有锁持有者处理文件任务
isOwner = await tryAcquireSchedulerLock(lockOpts)

// 非持有者定期探测锁（5s 间隔）
lockProbeTimer = setInterval(() => {
  void tryAcquireSchedulerLock(lockOpts).then(owned => {
    if (owned) { isOwner = true; ... }
  })
}, LOCK_PROBE_INTERVAL_MS) // 5000ms
```

Session 级任务不需要锁——它们是进程私有的。

### 6.5 Jitter 与负载均衡

通过 GrowthBook 的 `tengu_kairos_cron_config` 实时推送 jitter 配置，防止大量 KAIROS 实例在整点齐发：

```typescript
// src/utils/cronJitterConfig.ts
// 每 5 分钟重新拉取 jitter 配置
// ops 可以在负载尖峰时加宽 jitter 窗口
getJitterConfig: getCronJitterConfig
```

---

## 7. 通信层 — SendUserMessage (Brief)

### 7.1 工具接口

```typescript
// src/tools/BriefTool/BriefTool.ts
inputSchema = z.strictObject({
  message: z.string()
    .describe('The message for the user. Supports markdown.'),
  attachments: z.array(z.string()).optional()
    .describe('File paths to attach — photos, diffs, logs.'),
  status: z.enum(['normal', 'proactive'])
    .describe("'proactive' when surfacing something unsolicited."),
})
```

- `message`：Markdown 格式消息
- `attachments`：附件路径（图片、diff、日志）
- `status`：标记消息意图——`normal`（回复用户）或 `proactive`（主动推送）

### 7.2 启用门控链

```
isBriefEnabled() 门控链:

  编译时: feature('KAIROS') || feature('KAIROS_BRIEF')
    │
    ├─ getKairosActive() === true → 直接启用（Assistant 模式）
    │
    └─ getUserMsgOptIn() === true → 检查 entitlement:
         ├─ getKairosActive() → yes
         ├─ CLAUDE_CODE_BRIEF env → yes (开发测试)
         └─ tengu_kairos_brief GB gate → yes/no (5分钟刷新)
```

- **Assistant 模式**：绕过 opt-in，因为系统提示词已硬编码 "you MUST use SendUserMessage"
- **手动模式**：需要显式 opt-in（`--brief` / `defaultView: 'chat'` / `/brief` 命令）
- **Kill Switch**：`tengu_kairos_brief` 关闭时，已 opt-in 的会话也会在 5 分钟内失效

### 7.3 主动 vs 被动消息路由

```typescript
logEvent('tengu_brief_send', {
  proactive: status === 'proactive',
  attachment_count: attachments?.length ?? 0,
})
```

下游系统（推送通知、分析）根据 `status` 区分：
- `normal`：回复用户请求 → 常规展示
- `proactive`：主动推送 → 可能触发系统级通知

---

## 8. 感知层

### 8.1 终端焦点检测

通过终端 ANSI escape sequence（焦点事件报告），Claude 感知用户是否在看屏幕：

```typescript
// src/screens/REPL.tsx:2776-2778
...((feature('PROACTIVE') || feature('KAIROS'))
  && proactiveModule?.isProactiveActive()
  && !terminalFocusRef.current
  ? { terminalFocus:
      'The terminal is unfocused — the user is not actively watching.' }
  : {})
```

当终端失焦时，`terminalFocus` 上下文字段被注入到用户上下文中。系统提示词据此指导 Claude：

- **Unfocused** → 全自主：做决策、探索、提交、push，仅暂停于不可逆操作
- **Focused** → 协作式：展示选项、大变更前请示、保持输出简洁

### 8.2 GitHub Webhook 订阅

```typescript
// src/tools.ts
const SubscribePRTool = feature('KAIROS_GITHUB_WEBHOOKS')
  ? require('./tools/SubscribePRTool/SubscribePRTool.js').SubscribePRTool
  : null
```

`SubscribePRTool` 允许 Claude 订阅 PR 事件（评论、状态变更、审查请求等）。外部事件作为 channel 消息注入对话流。

### 8.3 Channel 消息系统

```typescript
// src/constants/xml.ts
export const CHANNEL_MESSAGE_TAG = 'channel-message'
export const CHANNEL_TAG = 'channel'
```

Channel 系统通过 MCP 服务器接收外部推送消息。`--channels` 参数指定要注册的 MCP channel 服务器：

```typescript
// src/main.tsx:3844-3846
program.addOption(new Option('--channels <servers...>',
  'MCP servers whose channel notifications (inbound push) should
   register this session.'));
```

Channel 消息在 UI 中可见（让用户看到有消息到达），但不可编辑（原始 XML）。

### 8.4 推送通知

```typescript
// src/tools.ts
const PushNotificationTool =
  feature('KAIROS') || feature('KAIROS_PUSH_NOTIFICATION')
    ? require('./tools/PushNotificationTool/PushNotificationTool.js')
        .PushNotificationTool
    : null
```

系统级推送通知，允许 Claude 在用户不看终端时通过 OS 级别通知联系用户。

---

## 9. 记忆管理 — 日志式追加 + 夜间蒸馏

### 9.1 日志文件布局

```
~/.claude/projects/<cwd>/memory/
├── MEMORY.md                    ← 索引文件（/dream 生成）
├── topic-a.md                   ← 主题文件（/dream 生成）
├── topic-b.md
└── logs/                        ← KAIROS 追加日志
    └── 2026/
        └── 04/
            ├── 2026-04-01.md
            ├── 2026-04-02.md
            ├── 2026-04-03.md
            └── 2026-04-04.md   ← 今天的追加日志
```

```typescript
// src/memdir/paths.ts:246-250
export function getAutoMemDailyLogPath(date: Date = new Date()): string {
  const yyyy = date.getFullYear().toString()
  const mm = (date.getMonth() + 1).toString().padStart(2, '0')
  const dd = date.getDate().toString().padStart(2, '0')
  return join(getAutoMemPath(), 'logs', yyyy, mm, `${yyyy}-${mm}-${dd}.md`)
}
```

### 9.2 /dream — 夜间记忆整合

`/dream` 是一个 bundled skill，在 `feature('KAIROS') || feature('KAIROS_DREAM')` 下注册。通常由 assistant 的 permanent cron 任务在夜间触发。

**四阶段整合流程**（`src/services/autoDream/consolidationPrompt.ts`）：

```
Phase 1 — Orient (定位)
├── ls 记忆目录
├── 读取 MEMORY.md 了解当前索引
└── 浏览现有主题文件，避免重复

Phase 2 — Gather (采集)
├── 优先：logs/YYYY/MM/YYYY-MM-DD.md 日期日志
├── 其次：与代码库现状矛盾的旧记忆
└── 补充：grep JSONL 转录中的关键信息

Phase 3 — Consolidate (整合)
├── 合并新信号到现有主题文件
├── 将相对日期转为绝对日期
└── 删除被推翻的旧事实

Phase 4 — Prune and Index (裁剪索引)
├── 更新 MEMORY.md (≤指定行数, ≤25KB)
├── 每条索引一行 ≤150字符
├── 删除过时指针
└── 解决矛盾
```

### 9.3 与标准 Auto Memory 的区别

| 特性 | 标准 Auto Memory | KAIROS 模式 |
|------|-----------------|-------------|
| 写入方式 | 实时更新 MEMORY.md | 追加到日期日志文件 |
| 索引维护 | 每次写入时更新 | 由 /dream 夜间整合 |
| Team Memory | 支持 | 不支持（与追加范式不兼容） |
| 适用场景 | 短会话、交互式 | 长时间运行的守护进程 |

```typescript
// src/memdir/memdir.ts:427-438
// KAIROS daily-log mode takes precedence over TEAMMEM
if (feature('KAIROS') && autoEnabled && getKairosActive()) {
  logMemoryDirCounts(getAutoMemPath(), {
    memory_type: 'auto',
  })
  return buildAssistantDailyLogPrompt(skipIndex)
}
```

---

## 10. Agent 行为差异

### 10.1 强制异步执行

KAIROS 模式下子 Agent 被**强制异步运行**：

```typescript
// src/tools/AgentTool/AgentTool.tsx
const assistantForceAsync = feature('KAIROS')
  ? appState.kairosEnabled : false;

const shouldRunAsync =
  run_in_background === true
  || agent.background === true
  || isCoordinatorMode()
  || forceAsync
  || assistantForceAsync       // ← KAIROS 强制异步
  || isProactiveActive()
```

原因：KAIROS 模式的主循环需要持续响应 tick 和用户消息，同步子 Agent 会阻塞整个循环。

### 10.2 Bash/PowerShell 后台化

```typescript
// src/tools/BashTool/BashTool.tsx:976
if (feature('KAIROS') && getKairosActive()
    && isMainThread && !isBackgroundTasksDisabled
    && run_in_background !== true) {
  // 自动转为后台执行
}

// PowerShellTool 同理
// src/tools/PowerShellTool/PowerShellTool.tsx:833
```

在 KAIROS 主线程中，Shell 命令默认后台执行，避免阻塞 tick 循环。

### 10.3 团队预初始化

```typescript
// src/main.tsx:1082-1086
// Pre-seed an in-process team so Agent(name: "foo") spawns
// teammates without TeamCreate.
assistantTeamContext =
  await assistantModule.initializeAssistantTeam();
```

KAIROS 启动时预建一个进程内团队，使得 `Agent(name: "foo")` 能直接生成 Teammate，无需显式调用 `TeamCreateTool`。团队上下文在 `setup()` 之前注入，确保 teammateMode 快照捕获到正确状态。

---

## 11. 防护与安全机制

### 11.1 用户中断 → 暂停

```typescript
// src/screens/REPL.tsx — onCancel()
proactiveModule?.pauseProactive();
// tick 停止发送，用户重获控制权
// 用户下次提交输入时自动恢复
```

用户按 Esc 立即暂停主动循环，不是"取消当前操作"而是"暂停整个守护进程"。

### 11.2 API 错误阻断

```typescript
// 错误消息 → 阻断 tick
if (newMessage.isApiErrorMessage) {
  proactiveModule?.setContextBlocked(true);
}
// 成功响应 → 解除阻断
else if (newMessage.type === 'assistant') {
  proactiveModule?.setContextBlocked(false);
}
```

防止 tick → API Error → 新 tick → API Error 的失控循环（如 auth 失败、rate limit）。

### 11.3 Context 压缩恢复

上下文窗口满时自动触发 compact：
1. 压缩历史消息为摘要
2. 注入 "你之前在自主工作，继续" 指令
3. 解除 contextBlocked → tick 恢复

这确保了长时间运行的 KAIROS 会话不会因上下文溢出而中断。

### 11.4 远程 Kill Switch

```typescript
// src/tools/ScheduleCronTool/prompt.ts
export function isKairosCronEnabled(): boolean {
  return getFeatureValue_CACHED_WITH_REFRESH(
    'tengu_kairos_cron',
    false,
    5 * 60 * 1000, // 5 分钟刷新
  )
}

// src/utils/cronScheduler.ts — check() 每秒检查
if (isKilled?.()) return
```

运维可以通过 GrowthBook 远程关闭：
- `tengu_kairos` → 完全禁用 KAIROS
- `tengu_kairos_cron` → 禁用 cron 调度器
- `tengu_kairos_brief` → 禁用 Brief 工具
- `tengu_kairos_cron_durable` → 禁用持久化 cron

### 11.5 目录信任检查

```typescript
if (!checkHasTrustDialogAccepted()) {
  console.warn(chalk.yellow(
    'Assistant mode disabled: directory is not trusted.'
  ));
}
```

KAIROS 会读取 `.claude/agents/assistant.md` 作为系统提示词的一部分。如果目录未被用户明确信任，拒绝激活，防止恶意项目通过 assistant.md 注入提示词。

---

## 12. 完整生命周期

```
用户启动 claude (KAIROS 构建)
        │
        ▼
 ┌─── 激活检查 ───────────────────────────────────┐
 │  ① feature('KAIROS') === true (编译时)          │
 │  ② isAssistantMode() — .claude/agents/assistant.md │
 │  ③ checkHasTrustDialogAccepted() — 目录已信任？ │
 │  ④ kairosGate.isKairosEnabled() — GrowthBook    │
 │  ⑤ setKairosActive(true)                        │
 │  ⑥ opts.brief = true — 强制 Brief               │
 │  ⑦ initializeAssistantTeam() — 预建团队         │
 └─────────────────────────────────────────────────┘
        │
        ▼
 ┌─── 首次 Tick ────────────────────────────────────┐
 │  <tick>14:30:05</tick> 注入消息队列               │
 │  → Claude: "Hi! What would you like to work on?" │
 │  → 用户不回 → Sleep(60000)                       │
 │  → 用户: "Review PR #42"                         │
 └──────────────────────────────────────────────────┘
        │
        ▼
 ┌─── 主循环 ──────────────────────────────────────┐
 │                                                  │
 │  ┌─ tick 到达 ─────────────────────────────────┐│
 │  │  检查有无工作:                               ││
 │  │  ├─ 有: 读代码/跑测试/修bug/提交PR          ││
 │  │  │   └─ SendUserMessage(status: 'proactive') ││
 │  │  └─ 无: Sleep(duration_ms)                   ││
 │  │       └─ 醒来 → 新 tick                      ││
 │  └──────────────────────────────────────────────┘│
 │                                                  │
 │  ┌─ 用户消息到达 ──────────────────────────────┐│
 │  │  priority: 'now' → 中断当前操作              ││
 │  │  处理用户请求                                ││
 │  │  SendUserMessage(status: 'normal')           ││
 │  │  → 新 tick (继续循环)                        ││
 │  └──────────────────────────────────────────────┘│
 │                                                  │
 │  ┌─ Cron 定时任务触发 ────────────────────────┐ │
 │  │  check() 每 1s 执行                         │ │
 │  │  now >= nextFireAt → 将 prompt 入队          │ │
 │  │  下次 tick 处理该 prompt                     │ │
 │  │  ├─ agentId → 路由到指定 Teammate            │ │
 │  │  ├─ recurring → 重新计算 nextFireAt          │ │
 │  │  └─ one-shot → 删除任务                      │ │
 │  └──────────────────────────────────────────────┘│
 │                                                  │
 │  ┌─ Channel/Webhook 事件 ─────────────────────┐ │
 │  │  GitHub PR 评论 → channel-message 注入       │ │
 │  │  Claude 评估并处理                           │ │
 │  └──────────────────────────────────────────────┘│
 │                                                  │
 │  ┌─ 防护触发 ─────────────────────────────────┐ │
 │  │  用户 Esc → pauseProactive() → tick 停止    │ │
 │  │  API Error → contextBlocked = true           │ │
 │  │  Context 满 → auto compact → 恢复           │ │
 │  │  GB kill switch → isKilled() → 停止          │ │
 │  └──────────────────────────────────────────────┘│
 └──────────────────────────────────────────────────┘
        │
        ▼ (夜间)
 ┌─── /dream 记忆整合 ──────────────────────────────┐
 │  permanent cron 触发 /dream skill                 │
 │  ├─ 读取 logs/YYYY/MM/YYYY-MM-DD.md              │
 │  ├─ grep JSONL 转录查找关键信息                    │
 │  ├─ 合并/更新主题文件                              │
 │  ├─ 裁剪 MEMORY.md 索引 (≤25KB)                  │
 │  └─ 删除过时记忆                                   │
 └──────────────────────────────────────────────────┘
```

---

## 13. 架构总结 — 守护进程类比

KAIROS 本质上是将 LLM 包装为一个**智能守护进程**。下表展示其与传统 Unix 守护进程的对应关系：

| 传统守护进程概念 | KAIROS 对应实现 | 源码位置 |
|----------------|----------------|---------|
| 事件循环 (Event Loop) | tick 心跳循环 | `cli/print.ts`, `screens/REPL.tsx` |
| `epoll_wait` / `select` | SleepTool (可中断等待) | `tools/SleepTool/` |
| Crontab | `.claude/scheduled_tasks.json` + CronScheduler | `utils/cronScheduler.ts` |
| 日志系统 | 日期日志 + `/dream` 夜间整合 | `memdir/paths.ts`, `services/autoDream/` |
| Webhook Handler | `SubscribePRTool` + channel-message | `tools/SubscribePRTool/` |
| `SIGHUP` 重载 | GrowthBook 运行时配置刷新 | `services/analytics/growthbook.ts` |
| `SIGTERM` 停止 | 用户 Esc / API 错误阻断 / GB kill switch | `screens/REPL.tsx` |
| 健康检查 | `terminalFocus` 感知 + contextBlocked 状态 | `ink/hooks/use-terminal-focus.ts` |
| `systemd notify` | `SendUserMessage` (Brief) | `tools/BriefTool/` |
| 工作进程 (Worker) | 异步子 Agent / Teammate | `tools/AgentTool/` |
| 文件锁 (Flock) | Scheduler Lock (多会话协调) | `utils/cronTasksLock.ts` |
| PID 文件 | lockIdentity (sessionId / UUID) | `utils/cronScheduler.ts` |

**关键区别**：传统守护进程的"决策逻辑"是开发者预先编码的 if/else。KAIROS 的决策逻辑**由 LLM 在系统提示词引导下实时推理产生**——每次 tick，Claude 观察完整上下文（代码状态、用户消息、终端焦点、待处理事件、记忆），然后自主判断下一步行动。这赋予了它传统守护进程不具备的**理解力、判断力和适应性**。

---

> **文件列表**（KAIROS 核心源码）：
>
> | 模块 | 关键文件 |
> |------|---------|
> | 入口/激活 | `src/main.tsx`, `src/assistant/gate.js`, `src/assistant/index.js` |
> | 状态管理 | `src/bootstrap/state.ts`, `src/state/AppStateStore.ts` |
> | Tick 循环 | `src/proactive/index.js`, `src/proactive/useProactive.js` |
> | 系统提示词 | `src/constants/prompts.ts`, `src/utils/systemPrompt.ts` |
> | Sleep 工具 | `src/tools/SleepTool/prompt.ts` |
> | Brief 工具 | `src/tools/BriefTool/BriefTool.ts`, `src/tools/BriefTool/prompt.ts` |
> | Cron 调度 | `src/utils/cronScheduler.ts`, `src/utils/cronTasks.ts`, `src/utils/cronJitterConfig.ts` |
> | Cron 工具 | `src/tools/ScheduleCronTool/CronCreateTool.ts` |
> | /loop Skill | `src/skills/bundled/loop.ts` |
> | /dream Skill | `src/skills/bundled/dream.js`, `src/services/autoDream/consolidationPrompt.ts` |
> | 记忆路径 | `src/memdir/paths.ts`, `src/memdir/memdir.ts` |
> | GitHub Webhook | `src/tools/SubscribePRTool/` |
> | 推送通知 | `src/tools/PushNotificationTool/` |
> | Channel 消息 | `src/services/mcp/channelNotification.ts` |
> | 终端焦点 | `src/ink/hooks/use-terminal-focus.ts` |
> | 定时任务 Hook | `src/hooks/useScheduledTasks.ts` |
> | 会话历史 | `src/assistant/sessionHistory.ts` |
