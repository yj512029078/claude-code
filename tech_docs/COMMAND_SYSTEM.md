---
title: Claude Code 命令系统
aliases: [命令系统, Command System]
series: Claude Code 架构解析
category: 工具与Agent
order: 11
tags:
  - claude-code
  - commands
  - slash-commands
  - lazy-loading
date: 2026-04-04
---

# Claude Code 命令系统深度技术文档

## 1. 概览

Claude Code 的命令系统管理着 **70+ 斜杠命令**，是用户与 AI 交互的核心界面。系统的设计目标是：

- **统一类型**: 三种命令类型（prompt / local / local-jsx）用一个判别联合类型覆盖
- **懒加载**: 所有命令实现都是 `load()` 延迟加载，降低启动开销
- **多源合并**: 内建命令、技能、插件、工作流、MCP 共用同一套命令接口
- **声明式元数据**: 命令的可用性、启用状态、权限要求全部声明式

---

## 2. 类型系统 (`types/command.ts`)

### 2.1 判别联合: Command

```typescript
type Command = CommandBase & (PromptCommand | LocalCommand | LocalJSXCommand)
```

通过 `type` 字段区分三种命令变体，共享 `CommandBase` 公共字段。

### 2.2 CommandBase — 公共元数据

```typescript
type CommandBase = {
  name: string                    // 命令名 (如 "compact", "review")
  description: string             // 描述 (显示在帮助和自动补全中)
  aliases?: string[]              // 别名 (如 clear 的别名是 reset, new)
  availability?: CommandAvailability[]  // 认证要求 (见下文)
  isEnabled?: () => boolean       // 动态启用检查 (Feature Flag 等)
  isHidden?: boolean              // 是否隐藏 (不在自动补全中显示)
  argumentHint?: string           // 参数提示 (如 "[on|off]")
  whenToUse?: string              // 详细使用场景 (给模型看的)
  immediate?: boolean             // 是否立即执行 (跳过队列)
  isSensitive?: boolean           // 参数是否脱敏
  disableModelInvocation?: boolean  // 是否禁止模型调用
  userInvocable?: boolean         // 是否允许用户调用
  loadedFrom?: 'commands_DEPRECATED' | 'skills' | 'plugin'
             | 'managed' | 'bundled' | 'mcp'  // 来源标记
  kind?: 'workflow'               // 工作流标记
  hasUserSpecifiedDescription?: boolean  // 是否有用户指定的描述
  version?: string                // 版本号
  userFacingName?: () => string   // 显示名 (与 name 不同时)
}
```

### 2.3 三种命令类型

#### PromptCommand — 展开为模型输入

```typescript
type PromptCommand = {
  type: 'prompt'
  progressMessage: string          // 执行时的进度提示 (如 "reviewing pull request")
  contentLength: number            // 内容长度 (token 估算用)
  source: SettingSource | 'builtin' | 'mcp' | 'plugin' | 'bundled'
  allowedTools?: string[]          // 限制可用工具
  model?: string                   // 指定模型
  effort?: EffortValue             // 努力等级
  hooks?: HooksSettings            // 关联的钩子
  skillRoot?: string               // 技能资源根目录
  context?: 'inline' | 'fork'     // 执行上下文
  agent?: string                   // fork 时使用的 Agent 类型
  paths?: string[]                 // 条件激活的文件路径模式
  pluginInfo?: { ... }             // 插件信息

  getPromptForCommand(             // 核心: 生成发给模型的内容
    args: string,
    context: ToolUseContext,
  ): Promise<ContentBlockParam[]>
}
```

**执行方式**: 调用 `getPromptForCommand()` 生成内容块，注入到对话中发给模型。
**典型场景**: 技能 (skills)、代码审查 (/review)、自定义 prompt 命令。

#### LocalCommand — 本地执行，返回文本

```typescript
type LocalCommand = {
  type: 'local'
  supportsNonInteractive: boolean  // 是否支持非交互模式
  load: () => Promise<LocalCommandModule>  // 懒加载实现
}

type LocalCommandModule = {
  call: (args: string, context: LocalJSXCommandContext) => Promise<LocalCommandResult>
}

type LocalCommandResult =
  | { type: 'text'; value: string }                    // 文本输出
  | { type: 'compact'; compactionResult: CompactionResult }  // 压缩结果
  | { type: 'skip' }                                   // 无输出
```

**执行方式**: 调用 `load()` 加载模块，执行 `call()`，返回结构化结果。
**典型场景**: /clear, /compact, /vim (纯逻辑命令，无需渲染 UI)。

#### LocalJSXCommand — 本地执行，渲染 React UI

```typescript
type LocalJSXCommand = {
  type: 'local-jsx'
  load: () => Promise<LocalJSXCommandModule>  // 懒加载实现
}

type LocalJSXCommandModule = {
  call: (
    onDone: LocalJSXCommandOnDone,
    context: ToolUseContext & LocalJSXCommandContext,
    args: string,
  ) => Promise<React.ReactNode>
}

type LocalJSXCommandOnDone = (
  result?: string,
  options?: {
    display?: 'skip' | 'system' | 'user'
    shouldQuery?: boolean         // 完成后是否发给模型
    metaMessages?: string[]       // 隐藏的元消息
    nextInput?: string            // 链式执行下一条命令
    submitNextInput?: boolean     // 是否自动提交
  },
) => void
```

**执行方式**: 调用 `load()` 加载模块，执行 `call()`，返回 React 组件渲染到终端。
通过 `onDone` 回调控制命令完成后的行为。
**典型场景**: /config, /model, /help, /plan, /skills (需要交互式 UI 的命令)。

### 2.4 命令可用性 (Availability)

```typescript
type CommandAvailability =
  | 'claude-ai'   // Claude AI 订阅用户
  | 'console'     // Console API Key 用户

// availability 与 isEnabled 是两个维度:
// availability = 谁可以用 (认证/provider 要求，静态)
// isEnabled    = 现在是否开启 (Feature Flag, 平台检测, 环境变量)
```

---

## 3. 命令注册 (`commands.ts`)

### 3.1 三层命令来源

```
                 ┌─────────────────────────────┐
                 │    getCommands(cwd)          │
                 │    最终用户可用的命令列表       │
                 └────────────┬────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
   ┌──────────────┐  ┌───────────────┐  ┌──────────────┐
   │  扩展命令     │  │  内建命令      │  │  动态命令     │
   │  (异步加载)   │  │  (同步注册)    │  │  (运行时发现) │
   └──────┬───────┘  └───────┬───────┘  └──────┬───────┘
          │                  │                  │
   ┌──────┴───────┐  ┌──────┴───────┐  ┌──────┴───────┐
   │ bundledSkills │  │ COMMANDS()   │  │ dynamicSkills│
   │ builtinPlugin │  │ 70+ 内建命令  │  │ 条件技能     │
   │ skillDirCmds  │  │              │  │ 嵌套目录发现  │
   │ workflowCmds  │  │              │  │              │
   │ pluginCmds    │  │              │  │              │
   │ pluginSkills  │  │              │  │              │
   └──────────────┘  └──────────────┘  └──────────────┘
```

### 3.2 合并顺序

```typescript
// loadAllCommands() — 命令合并管道
const allCommands = [
  ...bundledSkills,        // 1. 内建技能 (编译时注册)
  ...builtinPluginSkills,  // 2. 内建插件技能
  ...skillDirCommands,     // 3. 用户目录技能 (.claude/skills/)
  ...workflowCommands,     // 4. 工作流命令
  ...pluginCommands,       // 5. 外部插件命令
  ...pluginSkills,         // 6. 外部插件技能
  ...COMMANDS(),           // 7. 内建斜杠命令 (最后)
]
```

扩展命令排在前面，内建命令排在最后。这影响了技能搜索时的展示顺序。

### 3.3 两级过滤

```typescript
export async function getCommands(cwd: string): Promise<Command[]> {
  const allCommands = await loadAllCommands(cwd)  // memoized
  const dynamicSkills = getDynamicSkills()

  // 过滤 1: 认证要求
  // 过滤 2: 启用检查
  const baseCommands = allCommands.filter(
    _ => meetsAvailabilityRequirement(_) && isCommandEnabled(_),
  )

  // 动态技能去重后插入 (在内建命令之前)
  // ...
}
```

**注意**: `loadAllCommands` 是 memoized 的（因为加载涉及磁盘 I/O），但 `getCommands` **每次都重新过滤**，因为认证状态可能在会话中改变（如 `/login`）。

### 3.4 缓存管理

```typescript
// 清除命令 memoization (不清除技能缓存)
clearCommandMemoizationCaches()

// 完整清除 (命令 + 插件 + 技能)
clearCommandsCache()

// 当动态技能被加载时调用
clearCommandMemoizationCaches()  // → 同时清 skillSearch 索引
```

---

## 4. 命令分发流 (Dispatch Flow)

### 4.1 完整路径: 用户输入 → 命令执行

```
用户输入: "/compact 保留关键信息"
    │
    ▼
[handlePromptSubmit] — 输入处理入口
    │
    ├── 特殊词重写:
    │   "exit" → "/exit"
    │   "quit" → "/exit"
    │   ":q"   → "/exit"
    │
    ├── immediate 命令快速路径:
    │   如果命令标记 immediate + 当前有查询在跑
    │   → 直接 load() + call()，不进队列
    │
    ├── 查询进行中?
    │   → enqueue 到消息队列 (优先级: next)
    │   → 等待当前查询完成后处理
    │
    └── 查询空闲
        → executeUserInput()
            │
            ▼
[processUserInput] — 输入解析与路由
    │
    ├── 以 "/" 开头 + 非 bridge 消息?
    │   → import('./processSlashCommand.js')
    │   → processSlashCommand()
    │
    └── 否则 → 作为普通文本发给模型
            │
            ▼
[processSlashCommand] — 斜杠命令处理
    │
    ├── parseSlashCommand(input)
    │   → 解析命令名和参数
    │   → 无效格式? → 返回帮助文本
    │
    ├── hasCommand(name, commands)?
    │   ├── 否 + 像技能名? → "Unknown skill" 错误
    │   ├── 否 + 像路径? → 作为普通文本发给模型
    │   └── 是 → getCommand(name, commands)
    │
    └── getMessagesForSlashCommand(command, args)
            │
            ▼
[按 type 分发]
    │
    ├── type === 'local-jsx':
    │   └── command.load()
    │       → mod.call(onDone, context, args)
    │       → 渲染 React 组件到终端
    │       → onDone 回调决定后续行为
    │
    ├── type === 'local':
    │   └── command.load()
    │       → mod.call(args, context)
    │       → 返回 LocalCommandResult
    │       ├── { type: 'text', value } → 显示文本
    │       ├── { type: 'compact', ... } → 压缩结果
    │       └── { type: 'skip' } → 无输出
    │
    └── type === 'prompt':
        ├── context === 'fork'?
        │   → executeForkedSlashCommand() → 子 Agent 执行
        └── context === 'inline'?
            → getPromptForCommand(args, ctx)
            → 内容注入对话 → shouldQuery: true
            → 发给模型处理
```

### 4.2 消息队列

```typescript
// messageQueueManager.ts — 三级优先级
type QueuePriority = 'now' | 'next' | 'later'

// now:   立即处理 (内部系统消息)
// next:  下一个处理 (用户输入的命令，默认优先级)
// later: 延后处理 (任务通知等)

enqueue(command: QueuedCommand, priority: QueuePriority = 'next')
dequeue()  // 按优先级出队
getCommandsByMaxPriority()  // 获取最高优先级的所有命令

isSlashCommand(cmd)  // 判断: 字符串 + 以 "/" 开头 + !skipSlashCommands
```

### 4.3 immediate 命令

某些命令标记 `immediate: true`，可以**绕过队列**，在当前查询运行时直接执行。

```typescript
// 始终 immediate:
exit   (immediate: true)
hooks  (immediate: true)

// 条件 immediate:
model  (immediate: shouldInferenceConfigCommandBeImmediate())
effort (immediate: shouldInferenceConfigCommandBeImmediate())
fast   (immediate: shouldInferenceConfigCommandBeImmediate())
```

这些命令通常是**模式切换**（改模型、改努力等级），不需要等当前查询结束。

---

## 5. 命令实现模式

### 5.1 最小化 index.ts 模式

每个命令的 `index.ts` 只做**元数据声明**，实现在另一个文件中懒加载：

```typescript
// commands/clear/index.ts — 只有元数据
import type { Command } from '../../commands.js'

const clear = {
  type: 'local',
  name: 'clear',
  description: 'Clear conversation history and free up context',
  aliases: ['reset', 'new'],
  supportsNonInteractive: false,
  load: () => import('./clear.js'),  // 懒加载实现
} satisfies Command

export default clear
```

**好处**:
- 命令注册时不加载实现模块 → 启动更快
- 元数据可以在 import 阶段静态获取
- `satisfies Command` 保证类型安全但不改变推断

### 5.2 动态描述模式

```typescript
// commands/model/index.ts — description 是 getter
export default {
  type: 'local-jsx',
  name: 'model',
  get description() {
    // 每次访问时动态计算，包含当前模型名
    return `Set the AI model (currently ${renderModelName(getMainLoopModel())})`
  },
  get immediate() {
    return shouldInferenceConfigCommandBeImmediate()
  },
  load: () => import('./model.js'),
} satisfies Command
```

**好处**: 描述中可以包含实时状态（当前模型、当前模式等），自动补全时用户看到最新信息。

### 5.3 双变体模式 (交互/非交互)

```typescript
// commands/context/index.ts — 同名命令的两个变体
export const context: Command = {
  name: 'context',
  type: 'local-jsx',
  isEnabled: () => !getIsNonInteractiveSession(),  // 交互模式启用
  load: () => import('./context.js'),
}

export const contextNonInteractive: Command = {
  type: 'local',
  name: 'context',
  supportsNonInteractive: true,
  isEnabled: () => getIsNonInteractiveSession(),   // 非交互模式启用
  get isHidden() { return !getIsNonInteractiveSession() },
  load: () => import('./context-noninteractive.js'),
}

// 同一个 /context 命令，根据环境自动切换实现
// 交互模式 → 渲染彩色网格 (JSX)
// SDK/CI 模式 → 输出纯文本
```

### 5.4 Prompt 命令模式

```typescript
// commands/review.ts — prompt 类型
const review: Command = {
  type: 'prompt',
  name: 'review',
  description: 'Review a pull request',
  progressMessage: 'reviewing pull request',
  contentLength: 0,
  source: 'builtin',

  async getPromptForCommand(args): Promise<ContentBlockParam[]> {
    return [{ type: 'text', text: `
      You are an expert code reviewer. Follow these steps:
      1. If no PR number is provided, run \`gh pr list\`
      2. If a PR number is provided, run \`gh pr view <number>\`
      3. Run \`gh pr diff <number>\` to get the diff
      4. Analyze the changes...
      PR number: ${args}
    ` }]
  },
}
```

**Prompt 命令的本质**: 将用户的 `/review 123` 转换为一段结构化的提示词注入对话，然后让模型去执行。

### 5.5 Feature Flag 条件加载

```typescript
// commands.ts — 编译时条件
const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null

const buddy = feature('BUDDY')
  ? require('./commands/buddy/index.js').default
  : null

// 注册时展开
const COMMANDS = memoize((): Command[] => [
  // 始终可用的命令
  clear, compact, config, ...

  // 条件命令 (null 被过滤)
  ...(voiceCommand ? [voiceCommand] : []),
  ...(buddy ? [buddy] : []),

  // 内部命令 (仅 Anthropic 员工)
  ...(process.env.USER_TYPE === 'ant' ? INTERNAL_ONLY_COMMANDS : []),
])
```

---

## 6. 技能如何成为命令

### 6.1 磁盘技能 → Command

```
.claude/skills/my-skill/SKILL.md
    │
    ▼
loadSkillsFromSkillsDir()
    │
    ├── 解析 YAML frontmatter:
    │   - name, description, allowed_tools
    │   - hooks, agent, effort, context
    │   - paths (条件激活)
    │
    ├── parseSkillFrontmatterFields()
    │
    └── createSkillCommand() → Command {
          type: 'prompt',
          name: 'my-skill',
          source: projectSettings,
          loadedFrom: 'skills',
          getPromptForCommand(args, ctx) {
            // 替换变量: ${CLAUDE_SKILL_DIR}, ${CLAUDE_SESSION_ID}
            // 执行内联 shell 命令
            // 前置 "Base directory" 行
            return [{ type: 'text', text: processedContent }]
          }
        }
```

### 6.2 内建技能 → Command

```typescript
// skills/bundled/index.ts 中注册
registerBundledSkill({
  name: 'create-rule',
  description: 'Create Cursor rules for persistent AI guidance',
  whenToUse: 'When you want to create a rule...',
  allowedTools: ['Bash', 'FileRead', 'FileWrite'],
  files: {
    'templates/rule-template.md': '...',
  },
  async getPromptForCommand(args, ctx) {
    return [{ type: 'text', text: '...' }]
  },
})

// registerBundledSkill 内部:
// 1. 如果有 files → 首次调用时安全解压到磁盘
// 2. 包装 getPromptForCommand 前置 "Base directory" 行
// 3. 构造 Command 对象加入 bundledSkills 数组
```

### 6.3 MCP 技能 → Command

```
MCP Server listPrompts()
    │
    ▼
MCP Client → createSkillCommand()
    │
    ├── source: 'mcp'
    ├── loadedFrom: 'mcp'
    └── getPromptForCommand → 通过 MCP 协议获取 prompt 内容
```

### 6.4 动态技能发现

```
用户编辑文件 src/subproject/foo.ts
    │
    ▼
discoverSkillDirsForPaths(['src/subproject/foo.ts'])
    │
    ├── 从 src/subproject/ 向上遍历
    ├── 发现 src/subproject/.claude/skills/
    ├── 检查 gitignore
    └── addSkillDirectories() → 加载为 dynamicSkills
         │
         ▼
clearCommandMemoizationCaches()
    → 下次 getCommands() 包含新技能

// 条件技能: 
// 技能 frontmatter 声明 paths: ["src/**/*.test.ts"]
// 只有编辑测试文件时才激活
activateConditionalSkillsForPaths(touchedPaths)
```

---

## 7. 命令安全控制

### 7.1 可用性门控

```typescript
meetsAvailabilityRequirement(cmd): boolean
// 'claude-ai' → 需要 Claude AI 订阅
// 'console' → 需要 Console API Key (非 3P, 非自定义 URL)
// 无 availability → 所有人可用

// 每次 getCommands() 重新评估
// 因为 auth 状态可能在会话中改变 (如 /login 后)
```

### 7.2 远程/Bridge 安全

```typescript
// 远程模式: 只允许安全命令
REMOTE_SAFE_COMMANDS = new Set([
  session, exit, clear, help, theme, color,
  vim, cost, usage, copy, btw, feedback,
  plan, keybindings, statusline, stickers, mobile,
])

// Bridge 模式: 按类型过滤
isBridgeSafeCommand(cmd):
  'local-jsx' → false (渲染本地 Ink UI，不安全)
  'prompt'    → true  (展开为文本，安全)
  'local'     → 需要在 BRIDGE_SAFE_COMMANDS 白名单中

BRIDGE_SAFE_COMMANDS = new Set([
  compact, clear, cost, summary, releaseNotes, files,
])
```

### 7.3 内部命令隔离

```typescript
INTERNAL_ONLY_COMMANDS = [
  backfillSessions, breakCache, bughunter,
  commit, commitPushPr, ctx_viz, goodClaude,
  issue, initVerifiers, mockLimits, ...
]

// 仅在以下条件同时满足时注册:
// 1. process.env.USER_TYPE === 'ant' (内部构建)
// 2. !process.env.IS_DEMO (非演示模式)
```

### 7.4 模型调用控制

```typescript
// 控制模型是否能通过 SkillTool 调用某个命令
disableModelInvocation: true  // 模型不能调用
disableModelInvocation: false // 模型可以调用 (默认)

// getSkillToolCommands() 过滤逻辑:
// 必须是 prompt 类型
// disableModelInvocation !== true
// source !== 'builtin'
// 且有描述或 whenToUse
```

---

## 8. onDone 回调: 命令结果的路由

`local-jsx` 命令的 `onDone` 回调是控制命令完成后行为的关键：

```typescript
type LocalJSXCommandOnDone = (
  result?: string,           // 可选的文本结果
  options?: {
    display?: 'skip' | 'system' | 'user'  // 结果如何展示
    shouldQuery?: boolean     // 是否继续发给模型
    metaMessages?: string[]   // 隐藏元消息 (模型可见，用户不可见)
    nextInput?: string        // 链式命令
    submitNextInput?: boolean // 自动提交链式命令
  },
) => void
```

### 典型用法

```typescript
// 模式 1: 纯 UI 操作，不需要模型参与
onDone()  // 无参调用，命令完成

// 模式 2: 结果显示给用户
onDone('Vim mode enabled', { display: 'system' })

// 模式 3: 结果需要模型处理
onDone('Here is the context data...', { shouldQuery: true })

// 模式 4: 链式命令
onDone('Model changed to opus', {
  nextInput: '/compact',         // 换模型后自动压缩
  submitNextInput: true,
})

// 模式 5: 隐藏上下文注入
onDone('', {
  shouldQuery: true,
  metaMessages: ['System context that the model needs but user doesn\'t see'],
})
```

---

## 9. 命令查找算法

```typescript
export function findCommand(
  commandName: string,
  commands: Command[],
): Command | undefined {
  return commands.find(
    _ =>
      _.name === commandName ||              // 精确匹配名称
      getCommandName(_) === commandName ||   // 匹配显示名
      _.aliases?.includes(commandName),      // 匹配别名
  )
}

// 例: 用户输入 "/reset"
// → name: "clear" ✗
// → aliases: ["reset", "new"] ✓
// → 匹配到 clear 命令
```

### 描述格式化 (UI 展示)

```typescript
formatDescriptionWithSource(cmd):
  workflow    → "描述 (workflow)"
  plugin     → "(插件名) 描述" 或 "描述 (plugin)"
  bundled    → "描述 (bundled)"
  builtin    → "描述" (不加标注)
  mcp        → "描述" (不加标注)
  其他       → "描述 (来源名)"
```

---

## 10. 生命周期与观测

```typescript
// commandLifecycle.ts — 极简的生命周期通知
type CommandLifecycleState = 'started' | 'completed'

setCommandLifecycleListener(callback)   // 注册监听器
notifyCommandLifecycle(uuid, state)     // 发射事件

// 用于:
// - query.ts: 查询开始/结束
// - structuredIO.ts: 结构化输出
// - remoteIO.ts: 远程控制
// 不参与斜杠命令解析，是上层查询/CLI 的生命周期
```

---

## 11. 设计模式总结

### 11.1 命令即数据

```
命令 = 声明式元数据 (index.ts) + 懒加载实现 (模块)

分离带来:
✓ 启动时只解析元数据，不加载实现
✓ 类型系统确保元数据完整
✓ 同一个 Command 接口统一了技能/插件/内建/MCP
```

### 11.2 三种类型覆盖所有场景

```
prompt    → 给模型用的 (技能/审查/分析)
local     → 纯逻辑的 (vim/clear/compact)
local-jsx → 需要 UI 的 (config/model/plan/help)

每种类型的加载和执行路径清晰分离，
在 processSlashCommand 中用 switch(type) 分发
```

### 11.3 多源统一

```
7 种来源 (bundled → builtin-plugin → skills-dir → workflow → plugin → plugin-skill → builtin)
统一为 Command[]
统一的过滤 (availability + isEnabled)
统一的查找 (findCommand)
统一的分发 (processSlashCommand)
```

### 11.4 渐进式发现

```
静态注册:  COMMANDS() — 启动时注册
异步加载:  loadAllCommands() — memoized, 首次 getCommands 触发
动态发现:  discoverSkillDirsForPaths() — 文件操作时触发
条件激活:  activateConditionalSkillsForPaths() — 路径匹配时激活

命令池在整个会话中不断增长，但缓存确保不重复加载
```

### 11.5 安全即分层

```
编译时: feature() 消除内部命令
注册时: USER_TYPE/IS_DEMO 条件
运行时: availability + isEnabled 过滤
分发时: bridge/remote 白名单
模型时: disableModelInvocation 控制
```
