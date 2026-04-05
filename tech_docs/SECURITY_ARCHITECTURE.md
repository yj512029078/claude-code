---
title: Claude Code 安全架构深度解析
aliases: [安全架构, Security Architecture]
series: Claude Code 架构解析
category: 全局架构
order: 3
tags:
  - claude-code
  - security
  - permissions
  - sandbox
  - defense-in-depth
date: 2026-04-04
---

# Claude Code 安全架构深度解析

## 概览

Claude Code 采用了**多层纵深防御 (Defense in Depth)** 的安全架构。从上到下共有 7 层安全机制协同工作，确保 AI Agent 在执行操作时始终受到约束。

```
┌─────────────────────────────────────────────┐
│         第 1 层: 组织策略限制 (Policy Limits)  │  ← 远程 API 下发的组织级策略
├─────────────────────────────────────────────┤
│         第 2 层: 权限模式 (Permission Modes)   │  ← 用户选择的全局行为模式
├─────────────────────────────────────────────┤
│         第 3 层: 声明式规则 (Permission Rules)  │  ← 多源配置的 allow/deny/ask 规则
├─────────────────────────────────────────────┤
│         第 4 层: 工具级检查 (Tool Permissions)  │  ← 每个工具自己的 checkPermissions
├─────────────────────────────────────────────┤
│         第 5 层: 文件系统安全 (Path Safety)     │  ← 路径校验、敏感目录保护
├─────────────────────────────────────────────┤
│         第 6 层: 运行时沙箱 (Sandbox)          │  ← OS 级 fs/network 隔离
├─────────────────────────────────────────────┤
│         第 7 层: 信息安全 (Info Security)       │  ← Undercover 模式、内部信息保护
└─────────────────────────────────────────────┘
```

---

## 第 1 层: 组织策略限制 (Policy Limits)

### 来源

`src/services/policyLimits/`

### 机制

组织管理员通过 Anthropic API (`/api/claude_code/policy_limits`) 下发策略，本地缓存在 `~/.claude/policy-limits.json`。

```typescript
// 策略数据结构
type PolicyLimits = {
  restrictions: Record<string, { allowed: boolean }>
}

// 运行时检查
isPolicyAllowed('allow_product_feedback') → boolean
```

### 特点

- **ETag 缓存** + 重试，减少 API 调用
- **Fail-Open 策略** — 大多数策略在缓存缺失时默认允许
- **例外**: `allow_product_feedback` 在紧急流量模式下 **Fail-Closed**（默认拒绝）
- **适用范围**: 仅首次方 API + Team/Enterprise OAuth 用户

### 影响

策略限制是**最顶层**的控制，可以从组织层面禁用整类功能，独立于用户本地的权限设置。

---

## 第 2 层: 权限模式 (Permission Modes)

### 来源

`src/types/permissions.ts`

### 五种外部模式

| 模式 | 行为 | 安全级别 |
|------|------|---------|
| `default` | 每次敏感操作都提示用户确认 | 最高 |
| `plan` | 只读规划模式，不执行修改操作 | 最高 |
| `acceptEdits` | 自动允许工作目录内的文件编辑，其他仍需确认 | 中等 |
| `bypassPermissions` | 跳过大部分权限检查（但 **不能** 跳过安全检查和明确的 ask 规则） | 较低 |
| `dontAsk` | 不提示用户，直接将所有需要确认的操作转为拒绝 | 特殊（适合无人值守） |

### 两种内部模式

| 模式 | 行为 |
|------|------|
| `auto` | 使用 **Transcript Classifier**（AI 分类器）自动判断是否允许，连续 3 次或累计 20 次拒绝后回退到交互式提示 |
| `bubble` | Agent/Teammate 模式，将权限请求冒泡到父级 |

### 关键设计: bypassPermissions 的限制

```
即使在 bypassPermissions 模式下，以下检查仍然强制执行：

✗ 不能跳过: 整工具 deny 规则 (步骤 1a)
✗ 不能跳过: 内容级 ask 规则 (步骤 1f) — 如 Bash(npm publish:*)
✗ 不能跳过: 安全检查 safetyCheck (步骤 1g) — 如 .git/、.claude/ 目录
✗ 不能跳过: 工具返回的 deny (步骤 1d)

✓ 可以跳过: 普通的 passthrough → ask
✓ 可以跳过: 工具返回的普通 ask（非规则、非安全检查）
```

这意味着**即便用户选择了最宽松的模式，核心安全边界依然不可绕过**。

---

## 第 3 层: 声明式权限规则 (Permission Rules)

### 来源

`src/utils/permissions/permissions.ts` (~1,487 行)

### 规则结构

```typescript
type PermissionRule = {
  source: PermissionRuleSource    // 规则来源
  ruleBehavior: 'allow' | 'deny' | 'ask'  // 行为
  ruleValue: {
    toolName: string              // 匹配的工具名
    ruleContent?: string          // 可选的内容模式 (如 "git push")
  }
}

// 规则来源优先级（8 个来源）
type PermissionRuleSource =
  | 'policySettings'    // 组织策略（最高）
  | 'flagSettings'      // Feature Flag
  | 'userSettings'      // 用户全局设置
  | 'projectSettings'   // 项目级设置
  | 'localSettings'     // 本地设置
  | 'cliArg'            // CLI 参数
  | 'command'           // 命令级
  | 'session'           // 当前会话
```

### 规则类型

#### 整工具规则 (Whole-tool rules)

```typescript
// 完全禁止使用 BashTool
{ toolName: 'Bash', ruleBehavior: 'deny' }

// 允许所有 MCP 服务器 "github" 的工具
{ toolName: 'mcp__github', ruleBehavior: 'allow' }
// 匹配: mcp__github__create_issue, mcp__github__list_repos, ...

// 通配符匹配
{ toolName: 'mcp__github__*', ruleBehavior: 'deny' }
```

#### 内容级规则 (Content-specific rules)

```typescript
// 禁止 Bash 执行 rm -rf
{ toolName: 'Bash', ruleContent: 'rm -rf', ruleBehavior: 'deny' }

// 允许 Bash 执行 git 命令
{ toolName: 'Bash', ruleContent: 'git *', ruleBehavior: 'allow' }

// 对 npm publish 始终询问
{ toolName: 'Bash', ruleContent: 'npm publish:*', ruleBehavior: 'ask' }
```

#### 文件路径规则

```typescript
// 允许编辑 src/ 目录下的文件
{ toolName: 'Edit', ruleContent: 'src/**', ruleBehavior: 'allow' }

// 禁止读取 .env 文件
{ toolName: 'Read', ruleContent: '**/.env*', ruleBehavior: 'deny' }
```

### MCP 工具的特殊匹配

```typescript
// toolMatchesRule 的 MCP 匹配逻辑:
// 规则 "mcp__server" 匹配该 server 下所有工具
// 规则 "mcp__server__tool" 精确匹配一个工具
// 规则 "mcp__server__*" 通配符匹配

function toolMatchesRule(tool, rule): boolean {
  // 对于 MCP 工具: mcp__{serverName}__{toolName}
  // 规则可以只写 server 前缀来匹配整个 server
}
```

### 企业级管控

```typescript
// allowManagedPermissionRulesOnly = true 时
// 清除所有非管控源的规则，只保留 policy/flag 级别
syncPermissionRulesFromDisk() {
  if (shouldAllowManagedPermissionRulesOnly()) {
    // 清空 user/project/local/cli/session 规则
    // 只保留 policySettings 和 flagSettings
  }
}
```

---

## 第 4 层: 工具级权限检查 (Tool-specific Permissions)

### 核心权限判定流水线

`hasPermissionsToUseToolInner` 是中心判定函数，按严格顺序执行：

```
hasPermissionsToUseToolInner(tool, input, context)
│
├── 步骤 1a: getDenyRuleForTool() → 整工具 deny?
│   └── 是 → 立即 DENY（不可绕过）
│
├── 步骤 1b: getAskRuleForTool() → 整工具 ask?
│   ├── Bash + 沙箱自动允许? → 跳过，交给 Bash 自己处理
│   └── 否 → ASK（需用户确认）
│
├── 步骤 1c: tool.checkPermissions(input, context)
│   └── 每个工具自己的语义检查
│
├── 步骤 1d: 工具返回 deny? → DENY（不可绕过）
│
├── 步骤 1e: 工具要求交互 + ask? → ASK（即使 bypass 也不跳过）
│
├── 步骤 1f: 内容级 ask 规则? → ASK（不可绕过）
│   └── 如 Bash(npm publish:*) 用户配置的明确 ask
│
├── 步骤 1g: safetyCheck? → ASK（不可绕过）
│   └── 如编辑 .git/、.claude/、.vscode/ 等敏感路径
│
├── 步骤 2a: bypassPermissions 模式? → ALLOW
│
├── 步骤 2b: 整工具 always-allow 规则? → ALLOW
│
└── 步骤 3: passthrough → 转为 ASK（需用户确认）
```

### 后处理层

`hasPermissionsToUseTool` 在内层判定后追加逻辑：

```
hasPermissionsToUseTool() 后处理:

├── dontAsk 模式:
│   └── 所有 ASK → 自动转为 DENY + "Permission was not granted"
│
├── auto 模式:
│   ├── 安全工具白名单? → 直接 ALLOW（如 FileRead）
│   ├── acceptEdits 快速路径:
│   │   └── 模拟 acceptEdits 模式重跑 checkPermissions → 如果 allow 则直接 ALLOW
│   ├── Transcript Classifier 分类:
│   │   ├── 分类器通过 → ALLOW
│   │   ├── 分类器拒绝 → 记录 denial + DENY
│   │   └── 分类器不可用 → "Iron Gate" 拒绝
│   └── 拒绝追踪:
│       ├── 连续 3 次拒绝 → 回退到交互式提示
│       └── 累计 20 次拒绝 → 回退到交互式提示
│
└── shouldAvoidPermissionPrompts (无头模式):
    └── ASK → DENY + 异步 Agent 原因
```

### BashTool 的深度安全检查 (~2,600 行)

`src/tools/BashTool/bashPermissions.ts` 是最复杂的工具级检查：

```
bashToolHasPermission(command, context)
│
├── 1. Tree-sitter AST 解析
│   ├── parseCommandRaw() → AST
│   ├── parseForSecurityFromAst() → 安全分析
│   └── 过于复杂? → 直接 ASK
│
├── 2. 沙箱自动允许路径
│   └── checkSandboxAutoAllow():
│       ├── 沙箱启用 + autoAllowBashIfSandboxed?
│       └── 仍然强制执行每个子命令的 deny/ask 规则
│
├── 3. 子命令拆分与逐一检查
│   ├── 管道符、&&、||、; 拆分
│   ├── 每个子命令独立检查:
│   │   ├── deny 规则匹配?
│   │   ├── ask 规则匹配?
│   │   ├── allow 规则匹配?
│   │   ├── 路径约束检查
│   │   └── 只读命令快速路径
│   └── cd + git 组合防护（bare-repo 类攻击）
│
├── 4. 命令去混淆
│   ├── stripSafeWrappers() — 去除 timeout、nice 等
│   ├── stripAllLeadingEnvVars() — 去除环境变量前缀
│   └── BINARY_HIJACK_VARS — 检测危险的 LD_PRELOAD 等
│
├── 5. 静态安全分析 (bashSecurity.ts)
│   ├── 命令替换检测 $()
│   ├── 进程替换检测 <()
│   ├── Zsh 特有问题
│   ├── 混淆检测 (IFS 篡改等)
│   ├── Git hook 注入检测
│   ├── sed 危险用法验证
│   └── 每种检测有唯一 ID 用于遥测
│
└── 6. Haiku 分类器 (可选)
    └── 对模糊的 deny/ask 描述使用小模型分类
```

### FileEdit/FileWrite 的权限检查

```
checkWritePermissionForTool(tool, input, context)
│
├── 1. Deny 规则 (gitignore 风格路径匹配)
├── 2. 内部可编辑路径 (plans, scratchpad)
├── 3. .claude/** 会话专用路径 (窄条件)
├── 4. checkPathSafetyForAutoEdit() — 安全检查:
│   ├── Windows 特殊路径 (ADS, 8.3名, UNC, 设备名)
│   ├── Claude 配置路径
│   └── "危险"目录/文件:
│       ├── .git/ 目录
│       ├── .vscode/ 目录
│       ├── Shell RC 文件 (.bashrc, .zshrc)
│       └── 其他敏感路径
├── 5. Ask 规则 (gitignore 风格)
├── 6. acceptEdits + 工作目录? → ALLOW
├── 7. Allow 规则 (gitignore 风格)
└── 8. 默认 → ASK + 生成建议
```

---

## 第 5 层: 文件系统安全 (Path Safety)

### 来源

`src/utils/permissions/filesystem.ts` (非常大的文件)

### 路径规范化

```typescript
// 所有路径操作都经过:
// 1. resolve() — 解析为绝对路径
// 2. 符号链接解析
// 3. 大小写规范化 (Windows/macOS)
// 4. 工作目录边界检查

pathInAllowedWorkingPath(filePath, context) → boolean
pathInWorkingPath(filePath, cwd) → boolean
```

### Windows 特殊防护

`checkPathSafetyForAutoEdit` 包含大量 Windows 路径攻击防护：

| 攻击向量 | 防护 |
|----------|------|
| Alternate Data Streams (ADS) | 检测 `:` 字符 |
| 8.3 短文件名 | 检测 `~` + 数字模式 |
| `\\?\` 前缀绕过 | 直接拒绝 |
| 尾部点/空格 | 检测并拒绝 |
| 设备名 (CON, PRN, NUL...) | 枚举检测 |
| UNC 路径 | 检测 `\\` 前缀 |

### 敏感路径保护

以下路径被视为"危险"，即使在 bypass 模式下也需要用户确认：

- `.git/` — Git 仓库内部
- `.vscode/` — VS Code 配置
- `.claude/` — Claude Code 配置
- `.bashrc`, `.zshrc`, `.profile` — Shell 配置
- `~/.ssh/` — SSH 密钥
- 其他系统级配置文件

### 路径规则匹配

使用 **gitignore 风格** 的模式匹配:

```typescript
matchingRuleForInput(filePath, context, toolType, behavior)
// 支持: *, **, ?, [abc] 等 glob 语法
// 每个设置源 (user/project/local) 有独立的根目录
```

---

## 第 6 层: 运行时沙箱 (Sandbox)

### 来源

- `src/tools/BashTool/shouldUseSandbox.ts` — 沙箱决策
- `src/utils/sandbox/sandbox-adapter.ts` — 沙箱运行时适配

### 沙箱决策逻辑

```typescript
shouldUseSandbox(input): boolean
│
├── SandboxManager 未启用? → false
├── dangerouslyDisableSandbox + 策略允许? → false
├── 无命令? → false
├── 命令在 excludedCommands 中? → false (非安全边界，仅 UX)
└── 其他 → true (使用沙箱)
```

### 排除命令的安全说明

代码中有明确注释：

```typescript
// NOTE: excludedCommands is a user-facing convenience feature, not a security boundary.
// It is not a security bug to be able to bypass excludedCommands — the sandbox permission
// system (which prompts users) is the actual security control.
```

即：`excludedCommands` 只是 **UX 便利**，真正的安全控制是权限系统（第 3-4 层），不是沙箱排除列表。

### 沙箱运行时配置

```typescript
convertToSandboxRuntimeConfig(settings):
│
├── 文件系统:
│   ├── 可写: cwd + temp 目录
│   ├── 只读: 其他已知安全路径
│   ├── 始终拒绝写入: settings 路径 (防止工具修改自己的权限)
│   └── WebFetch 域名规则合并
│
├── 网络:
│   ├── 默认: 限制为已知安全域名
│   └── allowManagedDomainsOnly: 仅策略指定的域名
│
└── 沙箱 ask 回调:
    └── 受管域名策略整合
```

### 双层防御

沙箱和权限系统是**两个独立层**，互补而非替代：

```
权限层: "你可以执行这个命令吗?" → 语义级别判断
沙箱层: "执行时能访问什么?" → OS 级别隔离

两者同时生效:
1. 权限层先判断是否允许执行
2. 如果允许，沙箱层限制执行时的能力范围
```

---

## 第 7 层: 信息安全 (Information Security)

### Undercover 模式

`src/utils/undercover.ts`

Anthropic 内部员工使用 Claude Code 向公开仓库贡献代码时激活：

```typescript
isUndercover() → boolean   // 根据环境或仓库类型判断
getUndercoverInstructions() → string  // 返回保密指令

// 防护内容:
// 1. 屏蔽内部模型代号 (Capybara, Tengu 等)
// 2. 不透露用户是 AI
// 3. 不泄露内部工具/系统信息
// 4. 仅在 USER_TYPE === 'ant' 的内部构建中存在
```

### 死代码消除

所有内部功能通过 `feature()` 编译时消除：

```typescript
// 外部构建中，这些代码完全不存在:
if (feature('ABLATION_BASELINE')) { ... }  // → 消除
process.env.USER_TYPE === 'ant'            // → false，分支消除
```

---

## Auto 模式的分类器安全

### Transcript Classifier

`src/utils/permissions/yoloClassifier.ts`

auto 模式使用 AI 分类器自动判断是否允许操作：

```
classifyYoloAction(tool, input, transcript)
│
├── 安全工具白名单 → 直接通过 (如 FileRead, Grep, Glob)
├── acceptEdits 快速路径 → 模拟 acceptEdits 判定
├── 分类器判断:
│   ├── ALLOW → 允许执行
│   ├── DENY → 拒绝并记录
│   └── 不可用 → "Iron Gate" 强制拒绝
└── 拒绝追踪 (Denial Tracking)
```

### 拒绝追踪 (Denial Tracking)

`src/utils/permissions/denialTracking.ts` — 防止分类器进入无限拒绝循环：

```typescript
const DENIAL_LIMITS = {
  maxConsecutive: 3,   // 连续 3 次拒绝
  maxTotal: 20,        // 累计 20 次拒绝
}

// 触发任一限制 → 回退到交互式提示
shouldFallbackToPrompting(state): boolean {
  return state.consecutiveDenials >= 3 || state.totalDenials >= 20
}
```

这确保了即使分类器持续拒绝，系统也不会卡死，而是让用户介入。

---

## Agent/子工具的安全隔离

### 工具集限制

`src/constants/tools.ts` 定义了不同角色可用的工具集：

```typescript
// 所有 Agent 禁止使用的工具
ALL_AGENT_DISALLOWED_TOOLS = [TaskOutput, PlanTools, AskUserQuestion, TaskStop, ...]

// 异步 Agent 允许的工具 (白名单)
ASYNC_AGENT_ALLOWED_TOOLS = [Read, Grep, FileEdit, FileWrite, Bash, ...]

// Coordinator 模式专用工具
COORDINATOR_MODE_ALLOWED_TOOLS = [Agent, TaskStop, SendMessage]
```

### 模型可见工具过滤

```typescript
// tools.ts 中 getTools() 的过滤链:
1. getAllBaseTools() — 获取所有基础工具
2. filterToolsByDenyRules() — 根据 deny 规则移除工具 (模型看不到)
3. isEnabled() — 工具自身启用检查
4. assembleToolPool() — 合并 MCP 工具，排重

// deny 规则在模型看到工具之前就生效:
// 如果 deny rule 匹配 mcp__server，该 server 的所有工具
// 在模型 prompt 中就不会出现
```

### Bridge/Remote 安全

```typescript
// 远程模式: 只允许安全命令
REMOTE_SAFE_COMMANDS = [session, exit, clear, help, theme, ...]

// Bridge 模式: prompt 类型命令通过，local-jsx 类型阻止
isBridgeSafeCommand(cmd):
  - 'local-jsx' → false (渲染 Ink UI，不安全)
  - 'prompt'   → true  (展开为文本，安全)
  - 'local'    → BRIDGE_SAFE_COMMANDS 白名单
```

---

## 完整安全检查流 (End-to-End)

```
模型请求调用工具
    │
    ▼
[第 1 层] isPolicyAllowed() — 组织策略检查
    │
    ▼
[第 3 层] getDenyRuleForTool() — 整工具 deny 规则?
    │ 是 → 拒绝 (模型甚至看不到这个工具)
    │
    ▼
[第 3 层] getAskRuleForTool() — 整工具 ask 规则?
    │
    ▼
[第 4 层] tool.checkPermissions() — 工具自身检查
    │ ├── BashTool: AST 解析 + 子命令拆分 + 安全分析
    │ ├── FileEditTool: 路径安全检查
    │ └── 其他工具: 各自的语义检查
    │
    ▼
[第 5 层] checkPathSafetyForAutoEdit() — 敏感路径?
    │ 即使 bypassPermissions 也不可跳过
    │
    ▼
[第 2 层] 权限模式判定:
    │ ├── bypassPermissions → 允许 (除了上面不可绕过的部分)
    │ ├── dontAsk → 将 ask 转为 deny
    │ ├── auto → 分类器判断 + 拒绝追踪
    │ └── default → 提示用户确认
    │
    ▼
[第 6 层] 如果允许执行:
    │ shouldUseSandbox(input)?
    │ ├── 是 → 在沙箱中执行 (fs/network 隔离)
    │ └── 否 → 直接执行
    │
    ▼
    执行完成，结果返回模型
```

---

## 设计原则总结

| 原则 | 体现 |
|------|------|
| **纵深防御** | 7 层独立安全机制，任何一层被绕过都有其他层兜底 |
| **最小权限** | Agent 只能看到和使用被授权的工具子集 |
| **不可绕过的安全检查** | safetyCheck 和内容级 ask 规则即使在 bypass 模式下也强制执行 |
| **Fail-Safe** | dontAsk 将不确定转为拒绝；分类器不可用时 "Iron Gate" 拒绝 |
| **拒绝熔断** | 连续/累计拒绝达到阈值时回退到人工介入 |
| **路径规范化** | 所有路径检查前先标准化，防止符号链接/大小写/特殊字符绕过 |
| **编译时隔离** | 内部功能通过 feature() 在外部构建中完全消除 |
| **声明式优先** | 权限规则是声明式的，来源可追溯，行为可预测 |
| **双层执行控制** | 权限层 (语义) + 沙箱层 (OS) 同时生效 |
