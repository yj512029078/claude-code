---
title: Claude Code 架构概览
aliases: [架构概览, Architecture Overview]
series: Claude Code 架构解析
category: 全局架构
order: 1
tags:
  - claude-code
  - architecture
  - typescript
  - react-ink
  - bun
date: 2026-04-04
---

# Claude Code 架构概览

## 1. 项目概述

### 1.1 项目来源

这是 **Anthropic 官方 AI 编程 CLI 工具 Claude Code** 的完整源代码，于 2026 年 3 月 31 日通过 npm 发布包中意外包含的 **sourcemap 文件** 泄露。Sourcemap 的 `sourcesContent` 字段嵌入了所有原始 TypeScript 源码。

### 1.2 项目定位

Claude Code 是一个**终端驻留的 AI 编程助手**，底层基于 Anthropic Claude 模型家族。它不是一个简单的 CLI wrapper，而是一个包含 **40+ 工具**、**多 Agent 编排**、**自定义终端 UI (React/Ink)**、**MCP 协议集成**、以及**多云 LLM Provider** 支持的大型复杂系统。

### 1.3 技术栈

| 层面 | 技术选型 |
|------|---------|
| **运行时** | Bun (首选) / Node.js 18+ |
| **语言** | TypeScript (严格模式) |
| **构建** | Bun bundler (`bun:bundle` feature flags 做死代码消除) |
| **终端 UI** | React + Ink (自定义终端渲染器) |
| **CLI 框架** | Commander.js (`@commander-js/extra-typings`) |
| **LLM SDK** | `@anthropic-ai/sdk` + Bedrock/Vertex/Foundry 变体 |
| **Schema 校验** | Zod v4 |
| **协议** | Model Context Protocol (MCP) |
| **工具库** | lodash-es, chalk, strip-ansi 等 |

---

## 2. 工程规模

```
src/
├── 1,902 个文件 (1,884 .ts/.tsx, 18 .js)
├── main.tsx              # 4,684 行，CLI 主入口
├── QueryEngine.ts        # ~1,296 行，核心对话引擎
├── query.ts              # ~1,730 行，Agentic 查询循环
├── commands.ts           # 755 行，命令注册中心
├── tools.ts              # 390 行，工具注册中心
├── Tool.ts               # ~793 行，工具基础类型定义
└── 30+ 顶级目录模块
```

---

## 3. 系统架构

### 3.1 启动流程 (Entry Pipeline)

```
cli.tsx (Bootstrap Entrypoint)
  │
  ├── 快速路径: --version/-v → 直接输出版本，零模块加载
  ├── 快速路径: --dump-system-prompt → 输出系统提示词
  ├── 特殊路径: --daemon-worker, bridge, remote-control 等
  │
  └── 正常路径 → import('../main.js') → cliMain()
        │
        ├── 1. 性能探针: profileCheckpoint('main_tsx_entry')
        ├── 2. 预取: MDM/Keychain 并行预读（macOS 优化）
        ├── 3. init(): 配置/Auth/遥测/OAuth/代理/策略
        ├── 4. GrowthBook: A/B 实验框架初始化
        ├── 5. Commander.js: 解析 CLI flags
        ├── 6. Trust Dialog: 首次使用信任确认
        ├── 7. MCP: 加载 MCP servers/tools/resources
        ├── 8. Commands: getCommands(cwd) 加载所有命令
        ├── 9. Tools: getTools() 组装工具池
        │
        └── 10. launchRepl() → 进入交互式 REPL
```

关键设计：`cli.tsx` 使用**全动态 `import()`** 确保快速路径（如 `--version`）几乎零开销。`main.tsx` 前几行就发起了 MDM 读取和 Keychain 预取的**并行副作用**，利用 import 阶段约 135ms 的模块求值时间做 I/O 重叠。

### 3.2 核心架构图

```
┌─────────────────────────────────────────────────────┐
│                    用户终端 (TTY)                     │
└────────────────────────┬────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────┐
│              React/Ink 终端 UI 层                     │
│  components/ │ hooks/ │ screens/ │ ink/ │ keybindings/│
└────────────────────────┬────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────┐
│                  REPL 交互层                          │
│  replLauncher.tsx │ interactiveHelpers.tsx            │
│  commands.ts (斜杠命令) │ dialogLaunchers.tsx         │
└────────────────────────┬────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────┐
│            QueryEngine (对话引擎核心)                  │
│  QueryEngine.ts: 会话状态、消息管理、Turn 循环         │
│  query.ts: Agentic 循环、Token 预算、压缩、工具执行    │
└────────────┬───────────────────────┬────────────────┘
             │                       │
┌────────────▼──────────┐ ┌─────────▼────────────────┐
│   Tool 执行层          │ │   LLM API 层              │
│  tools.ts (注册)       │ │  services/api/client.ts   │
│  Tool.ts (基类)        │ │  services/api/claude.ts   │
│  tools/* (40+ 工具)    │ │  utils/model/model.ts     │
│  services/tools/*      │ │  utils/model/providers.ts │
└────────────────────────┘ └──────────────────────────┘
             │                       │
┌────────────▼───────────────────────▼────────────────┐
│                 Services 服务层                       │
│  mcp/ │ oauth/ │ analytics/ │ autoDream/ │ compact/ │
│  policyLimits/ │ remoteManagedSettings/ │ plugins/   │
└─────────────────────────────────────────────────────┘
```

---

## 4. 核心模块详解

### 4.1 QueryEngine (`QueryEngine.ts`)

**职责**: 拥有完整对话生命周期和会话状态的核心类。

```typescript
export class QueryEngine {
  private config: QueryEngineConfig
  private mutableMessages: Message[]      // 会话消息存储
  private abortController: AbortController
  private permissionDenials: SDKPermissionDenial[]
  private totalUsage: NonNullableUsage    // token 用量追踪
  private readFileState: FileStateCache   // 文件状态缓存
  private discoveredSkillNames: Set<string>

  async *submitMessage(prompt, options): AsyncGenerator<SDKMessage>
}
```

核心概念：

- **每个对话一个 QueryEngine 实例**，`submitMessage()` 每次调用代表一个新的 Turn
- 状态（消息、文件缓存、用量）在多个 Turn 间持续保留
- 使用 **AsyncGenerator** 流式输出 SDK 消息
- 包裹 `canUseTool` 以追踪权限拒绝事件
- 支持 **snip replay**（历史裁剪重放，用于无头 SDK 模式的内存控制）

`QueryEngineConfig` 是其配置接口：

```typescript
type QueryEngineConfig = {
  cwd: string
  tools: Tools
  commands: Command[]
  mcpClients: MCPServerConnection[]
  agents: AgentDefinition[]
  canUseTool: CanUseToolFn
  getAppState: () => AppState
  setAppState: (f: (prev: AppState) => AppState) => void
  initialMessages?: Message[]
  readFileCache: FileStateCache
  customSystemPrompt?: string
  userSpecifiedModel?: string
  thinkingConfig?: ThinkingConfig
  maxTurns?: number
  maxBudgetUsd?: number
  // ...
}
```

### 4.2 Agentic 查询循环 (`query.ts`)

这是模型交互的**核心循环**，约 1,730 行，负责：

1. **Token 预算管理** — 计算剩余 token，决定是否触发压缩
2. **自动压缩 (Auto-Compact)** — 当上下文接近限制时自动触发，使用 `services/compact/` 下的压缩策略
3. **Reactive Compact** — 响应式压缩（Feature Flag 保护）
4. **Context Collapse** — 上下文折叠优化
5. **工具执行** — 通过 `StreamingToolExecutor` 和 `runTools` 执行模型请求的工具调用
6. **消息规范化** — `normalizeMessagesForAPI` 将内部消息格式转为 API 格式
7. **斜杠命令集成** — 识别并执行队列中的斜杠命令
8. **Skill 搜索预取** — 实验性功能
9. **Post-sampling Hooks** — 采样后钩子执行
10. **Analytics 事件** — 遥测与分析

关键依赖链：

```
query.ts
  → services/api/claude.ts    (HTTP 流式请求)
  → services/tools/StreamingToolExecutor.ts (流式工具执行)
  → services/tools/toolOrchestration.ts (工具编排)
  → services/compact/* (压缩策略)
  → utils/messages.ts (消息操作)
  → utils/tokens.ts (Token 计数)
```

### 4.3 LLM API 层

#### `services/api/client.ts` — API 客户端工厂

根据环境自动选择正确的 SDK 客户端：

```
getAnthropicClient()
  ├── AWS Bedrock  → AnthropicBedrock SDK
  ├── Azure Foundry → AnthropicFoundry SDK (+ Azure AD)
  ├── GCP Vertex    → AnthropicVertex SDK (+ GoogleAuth)
  ├── Claude.ai     → Anthropic SDK (OAuth token)
  └── Standard      → Anthropic SDK (API key)
```

自动注入：Session ID 头、User-Agent、自定义请求头、代理配置、超时设置。

#### `services/api/claude.ts` — 核心 API 交互层 (~3,400 行)

这是与 Claude API 交互的核心，关键导出：

| 函数 | 用途 |
|------|------|
| `queryModelWithStreaming` | 流式查询模型（主路径） |
| `queryModelWithoutStreaming` | 非流式查询 |
| `queryHaiku` | 快速小模型查询（用于辅助任务） |
| `queryWithModel` | 指定模型查询 |
| `buildSystemPromptBlocks` | 构建系统提示词块 |
| `addCacheBreakpoints` | 添加 Prompt 缓存断点 |
| `userMessageToMessageParam` | 消息格式转换 |
| `updateUsage` / `accumulateUsage` | 用量追踪 |
| `getMaxOutputTokensForModel` | 获取模型最大输出限制 |

#### `utils/model/providers.ts` — Provider 判断

```typescript
getAPIProvider() → 'firstParty' | 'bedrock' | 'vertex' | 'foundry'
isFirstPartyAnthropicBaseUrl() → boolean
```

被认证命令可用性、工具 Beta 特性剥离、API 客户端选择等场景引用。

#### `utils/model/model.ts` — 模型选择

```typescript
getMainLoopModel()        // 主循环使用的模型
getRuntimeMainLoopModel() // 运行时（含覆盖）的模型
getSmallFastModel()       // Haiku 等小型快速模型
getDefaultMainLoopModel() // 默认模型 (基于订阅类型)
parseUserSpecifiedModel() // 解析用户指定的模型字符串
```

---

## 5. 工具系统 (Tool System)

> **详细文档**: [TOOL_SYSTEM.md](./TOOL_SYSTEM.md)

Tool 系统采用**类型驱动的插件架构**，核心定义在 `src/Tool.ts`（~793 行）中。

### 5.1 架构概要

```
Tool 接口 (Tool.ts)  →  具体实现 (tools/*/)  →  注册表 (tools.ts)
```

- **`Tool<Input, Output, Progress>`**：三泛型参数的完整工具契约，包含 `call()`、`checkPermissions()`、`renderToolUseMessage()` 等 30+ 方法
- **`buildTool(def)`**：工厂函数，注入 fail-closed 安全默认值（`isConcurrencySafe→false`、`isReadOnly→false`）
- **`ToolUseContext`**：传递给 `call()` 的运行时上下文，包含 7 组字段（配置、运行控制、状态管理、UI 交互、执行追踪、Agent 信息、权限）
- **`ToolResult<T>`**：执行结果，可附带 `newMessages`（注入对话历史）和 `contextModifier`（修改后续上下文）

### 5.2 工具注册

`getAllBaseTools()` 是唯一注册点，包含 20+ 核心工具和 20+ Feature Flag 条件工具。通过 `getTools(permissionContext)` 过滤后交付模型使用。

### 5.3 权限系统

多层决策链：`validateInput` → `checkPermissions`(工具级) → `hasPermissionsToUseTool`(通用) → deny/allow/ask 规则 → Hook → Classifier → 用户交互。支持 7 种权限模式和 10+ 种决策原因类型。

### 5.4 核心特性

- **MCP 工具**：`MCPTool` 壳模板 + 运行时属性覆盖
- **Agent 定义**：三来源加载（built-in / Markdown 文件 / 插件），支持 tools 白/黑名单、model、effort、memory 等配置
- **ToolSearch 延迟加载**：`shouldDefer` + `searchHint` 减少 system prompt token 用量
- **进度上报**：`onProgress` 回调 + 7 种专属 Progress 类型
- **渲染方法族**：8 个生命周期渲染方法（调用、执行中、排队、拒绝、出错、完成、标签、分组）

---

## 6. 命令系统 (Command System)

### 6.1 命令类型

定义在 `types/command.ts` 中，使用判别联合类型：

```typescript
type Command = PromptCommand | LocalCommand | LocalJSXCommand

// PromptCommand — 生成文本发给模型
// LocalCommand — 本地执行，返回文本结果
// LocalJSXCommand — 本地执行，返回 React/Ink JSX
```

每个命令都有：`name`, `description`, `aliases?`, `availability?`, `isEnabled?`, `source`

### 6.2 命令注册 (`commands.ts`)

`commands.ts` 是命令注册中心，管理 **70+ 斜杠命令**。命令来源多样：

```
getCommands(cwd)
  ├── bundledSkills — 内建技能
  ├── builtinPluginSkills — 内建插件技能
  ├── skillDirCommands — 用户 /skills/ 目录
  ├── workflowCommands — 工作流脚本
  ├── pluginCommands — 外部插件
  ├── pluginSkills — 插件技能
  ├── dynamicSkills — 运行时动态发现的技能
  └── COMMANDS() — 内建斜杠命令
```

### 6.3 命令可用性过滤

```typescript
meetsAvailabilityRequirement(cmd) → boolean
// 基于认证状态动态判断：
// - 'claude-ai': 需要 Claude AI 订阅
// - 'console': 需要 Console API Key (排除3P)

// 每次 getCommands() 都重新评估（auth 可能在会话中改变）
```

### 6.4 安全命令集

```typescript
REMOTE_SAFE_COMMANDS  // 远程模式安全命令 (session, exit, clear, help, theme...)
BRIDGE_SAFE_COMMANDS  // Bridge 模式安全命令 (compact, clear, cost, summary...)
isBridgeSafeCommand() // 判断是否 bridge 安全
```

### 6.5 内部专用命令

```typescript
INTERNAL_ONLY_COMMANDS = [
  backfillSessions, breakCache, bughunter, commit, commitPushPr, ...
]
// 仅在 process.env.USER_TYPE === 'ant' 且非 demo 时加载
```

### 6.6 Feature Flag 条件命令

通过 Bun 的 `feature()` API 实现**编译时死代码消除**：

```typescript
const proactive = feature('PROACTIVE') || feature('KAIROS')
  ? require('./commands/proactive.js').default : null

const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default : null

const ultraplan = feature('ULTRAPLAN')
  ? require('./commands/ultraplan.js').default : null
```

这确保外部构建中不包含内部功能代码。

---

## 7. 服务层 (Services)

### 7.1 目录结构 (~130 文件)

```
services/
├── api/          # Claude API 交互 (client, claude, errors, bootstrap, logging)
├── analytics/    # 遥测、GrowthBook A/B 测试
├── autoDream/    # "梦境" 记忆整合系统
├── compact/      # 上下文压缩 (auto, reactive, snip)
├── contextCollapse/ # 上下文折叠
├── mcp/          # Model Context Protocol 服务
├── oauth/        # OAuth 认证流
├── policyLimits/ # 策略限制
├── plugins/      # 插件管理
├── remoteManagedSettings/ # 远程配置
├── skillSearch/  # 技能搜索索引
├── tools/        # 工具编排 (StreamingToolExecutor, toolOrchestration)
└── toolUseSummary/ # 工具使用摘要生成
```

### 7.2 MCP (Model Context Protocol) 集成

`services/mcp/` 是一个完整的 MCP 客户端实现：

- **client.ts** — MCP Server 生命周期管理，工具/资源发现
- **types.ts** — `MCPServerConnection`, `ServerResource`, `McpServerConfig` 等类型
- 支持加载外部 MCP servers 的 tools/resources/prompts
- MCP skills 可以被模型直接调用

### 7.3 AutoDream 系统

`services/autoDream/` 实现了一个独特的**记忆整合**机制：

1. **Orient** — 读取 `MEMORY.md`
2. **Gather** — 从日常日志中搜集新信号
3. **Consolidate** — 更新持久化记忆文件
4. **Prune** — 保持上下文精简

这模仿了人类睡眠中的记忆巩固过程，在后台 sub-agent 中运行。

### 7.4 上下文压缩策略

```
services/compact/
├── autoCompact.ts     # 自动压缩：token 快用完时触发
├── compact.ts         # 基础压缩：构建压缩后消息
├── reactiveCompact.ts # 响应式压缩 (Feature Flag)
└── snipCompact.ts     # 历史裁剪压缩 (Feature Flag)
```

---

## 8. 特色功能

### 8.1 BUDDY 系统 (Terminal Tamagotchi)

`src/buddy/` 是一个完整的**电子宠物系统**：

- 使用 Mulberry32 PRNG（基于 userId 种子）确定性抽卡
- 18 个物种，从 Common (Pebblecrab) 到 Legendary (Nebulynx)
- 每个 buddy 有 `DEBUGGING`, `CHAOS`, `SNARK` 等属性
- 拥有 Claude 生成的"灵魂描述"

### 8.2 Undercover 模式

`src/utils/undercover.ts` — Anthropic 员工使用 Claude Code 贡献公开代码仓库时：

- 屏蔽内部模型代号（如 *Capybara*, *Tengu*）
- 隐藏用户是 AI 的事实
- **Tengu** 被确认为 Claude Code 的内部代号

### 8.3 KAIROS — 主动式助手

Feature Flag `KAIROS` 下的"始终在线"模式：

- 监控日志并主动行动
- 不等待用户输入
- 支持推送通知、GitHub Webhook 订阅
- Brief 命令用于生成简报

### 8.4 ULTRAPLAN — 深度规划

Feature Flag `ULTRAPLAN` 下，将复杂任务卸载到远程 **Opus 4.6** 会话，可进行最长 30 分钟的深度规划。

### 8.5 Coordinator 模式 (Multi-Agent Swarm)

`src/coordinator/coordinatorMode.ts` — 多 Agent 编排：

- Coordinator 拥有 `AgentTool` + `TaskStopTool` + `SendMessageTool`
- Worker 拥有实际执行工具 (Bash/Read/Edit)
- 支持 Agent Swarms（`TeamCreate/DeleteTool`）
- 对等通信 (`ListPeersTool`, `UDS_INBOX`)

### 8.6 Bridge / Remote 模式

```
src/bridge/   # IDE 集成桥接
src/remote/   # 远程连接
```

- Bridge 模式允许从 IDE (JetBrains/VS Code) 控制
- Remote 模式允许从移动设备/Web 远程操作
- 通过 `BRIDGE_SAFE_COMMANDS` 和 `REMOTE_SAFE_COMMANDS` 控制安全边界

---

## 9. 状态管理

### 9.1 应用状态 (`state/AppState.ts`)

全局应用状态，包含 MCP 连接、命令列表、当前模式等。通过 `getAppState()` / `setAppState()` 函数式更新。

### 9.2 Bootstrap 状态 (`bootstrap/state.ts`)

早期启动状态：Session ID、原始 CWD、主线程 Agent 类型、模型覆盖等。这些在 init 阶段就需要确定。

### 9.3 文件状态缓存 (`utils/fileStateCache.ts`)

追踪已读取文件的状态，避免重复读取，跨 Turn 保留。

### 9.4 会话持久化

```typescript
flushSessionStorage()      // 刷新会话存储
recordTranscript()         // 记录对话转录
recordContentReplacement() // 记录内容替换
```

---

## 10. 构建系统与 Feature Flags

### 10.1 Bun Bundle Feature Flags

利用 Bun bundler 的 `feature()` 宏实现**编译时条件编译**：

```typescript
import { feature } from 'bun:bundle'

// 编译时求值 — 为 false 时整个分支被 tree-shaking 消除
if (feature('KAIROS')) { ... }
if (feature('ULTRAPLAN')) { ... }
if (feature('BUDDY')) { ... }
```

### 10.2 已知 Feature Flags

| Flag | 功能 |
|------|------|
| `PROACTIVE` | 主动模式 |
| `KAIROS` | KAIROS 主动助手 |
| `KAIROS_BRIEF` | KAIROS 简报 |
| `KAIROS_GITHUB_WEBHOOKS` | GitHub 事件订阅 |
| `KAIROS_PUSH_NOTIFICATION` | 推送通知 |
| `BRIDGE_MODE` | IDE 桥接 |
| `DAEMON` | 守护进程模式 |
| `VOICE_MODE` | 语音模式 |
| `HISTORY_SNIP` | 历史裁剪 |
| `WORKFLOW_SCRIPTS` | 工作流脚本 |
| `CCR_REMOTE_SETUP` | 远程设置 |
| `EXPERIMENTAL_SKILL_SEARCH` | 实验性技能搜索 |
| `ULTRAPLAN` | 深度规划 |
| `TORCH` | Torch 功能 |
| `UDS_INBOX` | Unix Domain Socket 通信 |
| `FORK_SUBAGENT` | Fork 子 Agent |
| `BUDDY` | 电子宠物 |
| `COORDINATOR_MODE` | 协调器模式 |
| `AGENT_TRIGGERS` | Agent 定时触发 |
| `AGENT_TRIGGERS_REMOTE` | 远程触发 |
| `MONITOR_TOOL` | 监控工具 |
| `WEB_BROWSER_TOOL` | 浏览器工具 |
| `CONTEXT_COLLAPSE` | 上下文折叠 |
| `REACTIVE_COMPACT` | 响应式压缩 |
| `OVERFLOW_TEST_TOOL` | 溢出测试 |
| `TERMINAL_PANEL` | 终端面板捕获 |
| `MCP_SKILLS` | MCP 技能 |
| `TEMPLATES` | 模板/任务分类 |
| `ABLATION_BASELINE` | 消融实验基线 |

### 10.3 环境变量条件

```typescript
process.env.USER_TYPE === 'ant'  // Anthropic 内部构建
process.env.IS_DEMO              // 演示模式
process.env.CLAUDE_CODE_SIMPLE   // 简单模式
process.env.CLAUDE_CODE_REMOTE   // 远程容器模式
process.env.NODE_ENV === 'test'  // 测试模式
```

---

## 11. 关键数据流

### 11.1 用户输入 → 模型响应 → 工具调用 循环

```
用户输入 (text/slash command)
    │
    ▼
processUserInput() — 解析用户输入
    │
    ├─ 斜杠命令 → findCommand() → 直接执行
    │
    └─ 自然语言 → QueryEngine.submitMessage()
         │
         ▼
    构建系统提示词 (fetchSystemPromptParts)
    加载记忆 (loadMemoryPrompt)
    准备上下文 (prependUserContext, appendSystemContext)
         │
         ▼
    query() — Agentic 循环
         │
         ├── queryModelWithStreaming() — 调用 Claude API
         │       │
         │       ▼
         │   流式响应 → StreamingToolExecutor
         │
         ├── 模型请求工具调用？
         │   └── 是 → runTools() → 执行工具 → 结果返回模型 → 继续循环
         │
         ├── Token 预算检查
         │   └── 接近限制 → autoCompact / reactiveCompact
         │
         ├── maxTurns 检查
         │   └── 超限 → 退出循环
         │
         └── 模型完成输出 → yield 消息 → 展示给用户
```

### 11.2 工具调用流

```
模型输出 tool_use block
    │
    ▼
StreamingToolExecutor — 流式接收工具参数
    │
    ▼
canUseTool() — 权限检查
    ├── deny 规则匹配 → 拒绝
    ├── 需要用户确认 → AskUserQuestionTool
    └── 允许 → 继续
    │
    ▼
tool.call(input, context) — 执行
    │
    ▼
ToolResultBlockParam — 返回结果
    │
    ▼
applyToolResultBudget() — 裁剪过大的结果
    │
    ▼
结果注入消息流 → 模型继续推理
```

---

## 12. 目录索引

| 目录 | 文件数 | 职责 |
|------|--------|------|
| `utils/` | ~564 | 最大模块：遥测、swarm、设置、shell、git、插件、权限、消息处理 |
| `components/` | ~389 | React/Ink UI 组件 |
| `commands/` | ~207 | 70+ 斜杠命令实现 |
| `tools/` | ~184 | 40+ Agent 工具实现 |
| `services/` | ~130 | 后台服务：MCP、OAuth、分析、AutoDream、压缩 |
| `hooks/` | ~104 | React/Ink Hooks |
| `ink/` | ~96 | 终端 UI 底层 |
| `bridge/` | ~31 | IDE 集成桥接 |
| `constants/` | ~21 | 常量定义 |
| `skills/` | ~20 | 技能系统 + 内建技能 |
| `cli/` | ~19 | CLI 管道、处理器、传输层 |
| `keybindings/` | ~14 | 键盘绑定 |
| `tasks/` | ~12 | 任务实现 (DreamTask, AgentTask) |
| `types/` | ~11 | 类型定义 + protobuf 生成类型 |
| `migrations/` | ~11 | 数据迁移 |
| `context/` | ~9 | 上下文管理 |
| `memdir/` | ~8 | 记忆目录助手 |
| `entrypoints/` | ~8 | 入口点 + SDK 入口 |
| `state/` | ~6 | 应用状态 |
| `buddy/` | ~6 | 电子宠物系统 |
| `vim/` | ~5 | Vim 模式绑定 |
| `coordinator/` | ~1 | 多 Agent 协调器 |
| `voice/` | ~1 | 语音模式 |

---

## 13. 设计亮点

### 13.1 性能优化

- **并行预取**: 启动时并行进行 MDM 读取、Keychain 预取、API 预连接
- **动态 import**: 快速路径零模块加载；按需加载重模块
- **Prompt 缓存**: `addCacheBreakpoints` + 工具按名称排序确保缓存稳定性
- **Memoization**: `lodash-es/memoize` 广泛用于命令/技能加载
- **Dead Code Elimination**: `feature()` 编译时消除外部构建中的内部代码

### 13.2 安全设计

- **Permission System**: 多层权限检查 (`canUseTool`, deny rules, filesystem sandbox)
- **Undercover Mode**: 防止 AI 身份和内部信息泄露
- **Bridge/Remote Safety**: 明确的命令白名单防止远程执行危险操作
- **Sandbox**: BashTool 支持沙箱模式 (`shouldUseSandbox.ts`)
- **Policy Limits**: 远程策略限制 + 本地速率限制

### 13.3 可扩展性

- **MCP Protocol**: 标准化的外部工具/资源/提示词集成
- **Plugin System**: 多级插件体系（内建、用户目录、外部）
- **Skill System**: 可发现、可调用的技能体系
- **Workflow System**: 自定义工作流脚本
- **Agent Definitions**: 自定义 Agent 配置 + 覆盖

### 13.4 容错设计

- `getSkills()` 中每个异步源都有独立的 `.catch()`，单个源失败不影响其他
- `getSlashCommandToolSkills` 返回空数组而不是抛出异常
- `clearCommandsCache()` 层级清理确保缓存一致性
- 所有工具执行结果经过 `applyToolResultBudget()` 裁剪防止溢出
