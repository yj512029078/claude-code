---
title: Claude Code Skill 系统
aliases: [Skill 系统, Skill System]
series: Claude Code 架构解析
category: 扩展生态
order: 15
tags:
  - claude-code
  - skills
  - markdown
  - extensibility
  - mcp-backed
date: 2026-04-04
---

# Claude Code Skill 系统深度技术文档

## 1. 概览

Skill 系统是 Claude Code 的**可扩展能力层**——一套将结构化 Markdown 文件转化为可被模型或用户调用的"技能"的机制。它的核心哲学是：

- **Markdown 即代码**: 一个 `SKILL.md` 文件就是一个完整的技能定义
- **多源发现**: managed (策略) → user (个人) → project (项目) → dynamic (运行时) → bundled (内置)
- **懒加载语义**: 列表中只暴露 name/description，完整 prompt 只在调用时展开
- **双路调用**: 用户 `/skill-name` 手动调用，或模型通过 `Skill` Tool 自动调用
- **安全隔离**: MCP 远程 skill 禁止 shell 执行，bundled skill 文件提取有路径遍历保护

> **与命令系统的关系**: Skill 是一种特殊的 `PromptCommand`（`type: 'prompt'`），通过 `loadedFrom` 字段与内建命令区分。Skill 最终合并到 `getCommands()` 返回的统一 `Command[]` 中，共享同一套分发、过滤、查找机制。命令系统的完整分析见 [[COMMAND_SYSTEM|命令系统]]。

---

## 2. Skill 定义格式

### 2.1 文件结构

每个 Skill 是一个 **目录 + `SKILL.md` 文件**的组合：

```
.claude/skills/
  deploy-staging/
    SKILL.md          ← 技能定义 (唯一必需文件)
    templates/         ← 可选: 附带的资源文件
      config.json
```

不支持单独的 `.md` 文件（Legacy `/commands/` 目录除外）。

### 2.2 SKILL.md 完整格式

```markdown
---
# === 身份与展示 ===
name: Deploy to Staging           # 显示名 (可选，默认用目录名)
description: Build and deploy     # 一行描述 (必需)
when_to_use: >                    # 详细使用场景 (给模型的提示，影响自动调用)
  Use when the user asks to deploy, push to staging,
  or mentions "ship it". Examples: "deploy main", "push to staging"
version: 1.0.0                    # 版本号 (可选)

# === 调用控制 ===
user-invocable: true              # 是否允许用户 /name 调用 (默认 true)
disable-model-invocation: false   # 是否禁止模型通过 Skill Tool 调用 (默认 false)
arguments:                        # 命名参数列表 (可选)
  - branch
  - environment
argument-hint: "<branch> [env]"   # 参数提示 (自动补全中显示)

# === 执行配置 ===
allowed-tools:                    # 允许的工具列表 (自动加入 alwaysAllow)
  - "Bash(npm:*)"
  - "Bash(git:*)"
  - Read
  - Write
model: sonnet                     # 模型覆盖 (可选，"inherit" = 不覆盖)
effort: high                      # 努力等级覆盖 (可选)
context: fork                     # 执行上下文: inline (默认) 或 fork (子 Agent)
agent: code-review                # fork 模式下使用的 Agent 类型
shell: bash                       # 内联 shell 命令 (!``) 的 shell 类型

# === 条件激活 ===
paths:                            # gitignore 风格的路径模式 (可选)
  - "src/frontend/**"             # 只在操作匹配文件时激活
  - "*.tsx"

# === 钩子 ===
hooks:
  PostToolUse:
    - matcher: Bash
      hooks:
        - command: "echo deployment complete"
---

# Deploy to Staging

这里是 skill 的 prompt 正文，会被完整注入到对话中。

## 变量替换

- `$ARGUMENTS` — 用户传入的完整参数字符串
- `$branch` / `${branch}` — 命名参数替换
- `${CLAUDE_SKILL_DIR}` — skill 目录的绝对路径
- `${CLAUDE_SESSION_ID}` — 当前会话 ID

## 内联 Shell 执行

行内: !`git status`
代码块:
​```!
npm run build
​```

注意: MCP 远程 skill 禁止执行内联 shell。
```

### 2.3 Frontmatter 解析 (`parseSkillFrontmatterFields`)

所有 frontmatter 字段由统一的解析函数处理（`src/skills/loadSkillsDir.ts:185-264`）：

```typescript
export function parseSkillFrontmatterFields(
  frontmatter: FrontmatterData,
  markdownContent: string,
  resolvedName: string,
): {
  displayName: string | undefined
  description: string                    // 无描述时从 Markdown 正文提取
  hasUserSpecifiedDescription: boolean
  allowedTools: string[]                 // 解析为工具名数组
  argumentHint: string | undefined
  argumentNames: string[]                // 解析 arguments 字段
  whenToUse: string | undefined
  version: string | undefined
  model: string | undefined              // parseUserSpecifiedModel() 处理
  disableModelInvocation: boolean
  userInvocable: boolean                 // 默认 true
  hooks: HooksSettings | undefined       // HooksSchema 验证
  executionContext: 'fork' | undefined
  agent: string | undefined
  effort: EffortValue | undefined        // 解析为数值或级别
  shell: FrontmatterShell | undefined
}
```

**关键设计**:
- `description` 如果 frontmatter 未指定，会从 Markdown 正文第一段自动提取
- `model: 'inherit'` 被解析为 `undefined`（不覆盖）
- `hooks` 使用 Zod schema 严格验证，无效配置被静默忽略并记录日志
- `paths` 使用 `ignore` 库（gitignore 语法）匹配

---

## 3. Skill 来源体系

### 3.1 六种来源

```typescript
// src/skills/loadSkillsDir.ts
export type LoadedFrom =
  | 'commands_DEPRECATED'   // Legacy: ~/.claude/commands/, .claude/commands/
  | 'skills'                // 磁盘: managed/user/project skills 目录
  | 'plugin'                // 外部插件提供的 skill
  | 'managed'               // 组织策略推送的 skill (MDM)
  | 'bundled'               // 编译进二进制的内置 skill
  | 'mcp'                   // MCP Server 提供的 prompt-type skill
```

### 3.2 磁盘 Skill 的路径映射

```typescript
// getSkillsPath() — 根据来源返回对应路径
function getSkillsPath(source: SettingSource, dir: 'skills'): string {
  switch (source) {
    case 'policySettings': return join(getManagedFilePath(), '.claude', 'skills')
    case 'userSettings':   return join(getClaudeConfigHomeDir(), 'skills')  // ~/.claude/skills/
    case 'projectSettings': return '.claude/skills'
  }
}
```

具体路径优先级（后加载的优先）：

```
┌─ 1. managed (组织策略) ── <managed_path>/.claude/skills/
│     • 由 MDM/策略推送
│     • CLAUDE_CODE_DISABLE_POLICY_SKILLS 可禁用
│
├─ 2. user (个人) ── ~/.claude/skills/
│     • 跟随用户，跨所有项目
│     • 受 isSettingSourceEnabled('userSettings') 控制
│
├─ 3. project (项目) ── .claude/skills/
│     • 沿目录树向上查找 (getProjectDirsUpToHome)
│     • 可被团队共享 (提交到 Git)
│     • 受 isSettingSourceEnabled('projectSettings') 控制
│
├─ 4. additional dirs ── --add-dir 参数指定的目录
│     • 每个目录的 .claude/skills/ 子目录
│     • bare 模式下是唯一的磁盘来源
│
└─ 5. legacy commands ── ~/.claude/commands/, .claude/commands/
      • 兼容旧版 SKILL.md / 单 .md 文件格式
      • skillsLocked 时跳过
```

### 3.3 Bundled Skills (内置技能)

Bundled Skill 是编译进二进制的技能，通过 `registerBundledSkill()` API 注册：

```typescript
// src/skills/bundledSkills.ts
export function registerBundledSkill(definition: BundledSkillDefinition): void {
  // 如果有 files → 构造 skillRoot，首次调用时安全解压到磁盘
  // 构造 Command 对象加入 bundledSkills 数组
}

// src/skills/bundled/index.ts — 启动时调用
export function initBundledSkills(): void {
  registerUpdateConfigSkill()
  registerKeybindingsSkill()
  registerVerifySkill()
  registerDebugSkill()
  registerLoremIpsumSkill()
  registerSkillifySkill()     // "用 skill 生成 skill" 的元技能
  registerRememberSkill()
  registerSimplifySkill()     // 并行 3-Agent 代码审查
  registerBatchSkill()
  registerStuckSkill()        // ant-only: 诊断卡住的会话
  // ... feature-gated skills
}
```

**当前内置 Skill 清单**:

| 名称 | 用途 | 可用范围 |
|------|------|---------|
| `update-config` | 更新 Claude Code 配置 | 所有用户 |
| `keybindings` | 管理快捷键绑定 | 所有用户 |
| `simplify` | 并行启动 3 个 Agent（复用/质量/效率）审查代码变更后自动修复 | 所有用户 |
| `batch` | 批量执行任务 | 所有用户 |
| `debug` | 启用 debug 日志并诊断会话问题 | 所有用户 |
| `lorem-ipsum` | 生成占位文本 (开发/测试用) | 所有用户 |
| `verify` | 运行应用验证代码变更是否正确 | ant-only |
| `remember` | 审查 auto-memory，提出迁移到 CLAUDE.md 的方案 | ant-only |
| `skillify` | 分析当前会话，通过多轮访谈自动生成 SKILL.md | ant-only |
| `stuck` | 诊断本机卡住/慢的 Claude Code 会话，发 Slack 报告 | ant-only |
| `dream` | 记忆巩固 (KAIROS feature) | feature-gated |
| `hunter` | Review artifact (REVIEW_ARTIFACT feature) | feature-gated |
| `loop` | 循环执行 (AGENT_TRIGGERS feature) | feature-gated |
| `claude-api` | Claude API 辅助 (BUILDING_CLAUDE_APPS feature) | feature-gated |

**Bundled Skill 的文件安全提取**:

带 `files` 属性的 bundled skill（如 `/verify`）需要在首次调用时把参考文件解压到磁盘：

```typescript
// 解压目录: getBundledSkillsRoot() + skill 名
// 包含 per-process nonce 防止预创建目录攻击
// O_NOFOLLOW | O_EXCL 防止符号链接攻击
// 0o700 目录权限 + 0o600 文件权限
// 路径遍历保护: normalized + 检查 ".." 和绝对路径
async function extractBundledSkillFiles(skillName: string, files: Record<string, string>)
```

### 3.4 MCP Skills

MCP Server 可以通过 `listPrompts()` 暴露 prompt-type skill：

```
MCP Server                        Claude Code
┌──────────────┐                ┌──────────────────┐
│ listPrompts  │ ──stdio/SSE─→ │ mcpSkillBuilders  │
│  name: "..."  │               │ → createSkillCommand()
│  desc: "..."  │               │ → loadedFrom: 'mcp'
│  getPrompt   │               │ → source: 'mcp'   │
└──────────────┘                └──────────────────┘
```

**安全限制**: MCP skill 的 Markdown 正文中的 `!` shell 命令（`!`cmd`` 和 ````!` 代码块）被**禁止执行**，因为远程 skill 内容不可信：

```typescript
// loadSkillsDir.ts:374
if (loadedFrom !== 'mcp') {
  finalContent = await executeShellCommandsInPrompt(finalContent, ...)
}
```

---

## 4. Skill 发现与加载

### 4.1 启动时加载

`getSkillDirCommands()` 是磁盘 skill 加载的核心入口，使用 `lodash.memoize` 缓存：

```
getSkillDirCommands(cwd)
  │
  ├── bare 模式？
  │   └── 是 → 只加载 --add-dir 的 skills，直接返回
  │
  ├── 并行加载 5 个来源:
  │   ├── managed skills (policySettings)
  │   ├── user skills (userSettings)
  │   ├── project skills (projectSettings, 沿目录树向上)
  │   ├── additional dir skills (--add-dir)
  │   └── legacy commands (commands_DEPRECATED)
  │
  ├── 合并 + 去重:
  │   └── getFileIdentity() → realpath 解析符号链接
  │       → 相同物理文件只保留首次加载的
  │
  ├── 分离条件 skill:
  │   ├── 无 paths 或已激活 → unconditionalSkills (立即可用)
  │   └── 有 paths 且未激活 → conditionalSkills (等待路径匹配)
  │
  └── 返回 unconditionalSkills
```

单个 skill 目录的加载流程：

```
loadSkillsFromSkillsDir(basePath, source)
  │
  ├── readdir(basePath) → 遍历子目录
  │
  ├── 每个子目录:
  │   ├── 非目录且非符号链接 → 跳过 (不支持单文件)
  │   ├── 读取 <dir>/SKILL.md
  │   │   └── ENOENT → 跳过; 其他错误 → 日志警告
  │   ├── parseFrontmatter(content, filePath)
  │   ├── parseSkillFrontmatterFields(frontmatter, content, name)
  │   ├── parseSkillPaths(frontmatter) → 条件路径
  │   └── createSkillCommand({...parsed, source, baseDir, loadedFrom: 'skills'})
  │
  └── 过滤 null → 返回 SkillWithPath[]
```

### 4.2 所有来源汇聚: `getCommands()`

```typescript
// commands.ts — loadAllCommands()
const allCommands = [
  ...bundledSkills,          // 1. initBundledSkills() 注册的内置 skill
  ...builtinPluginSkills,    // 2. 内建插件 skill
  ...skillDirCommands,       // 3. getSkillDirCommands() 的磁盘 skill
  ...workflowCommands,       // 4. 工作流命令
  ...pluginCommands,         // 5. 外部插件命令
  ...pluginSkills,           // 6. 外部插件 skill
  ...COMMANDS(),             // 7. 70+ 内建斜杠命令 (最后)
]

// getCommands() 每次调用:
// 1. 获取 memoized 的 allCommands
// 2. 获取 dynamicSkills (运行时发现的)
// 3. 两级过滤: meetsAvailabilityRequirement() + isCommandEnabled()
// 4. 动态 skill 去重后插入 (在内建命令之前)
```

### 4.3 运行时动态发现

当 `FileReadTool`、`FileWriteTool`、`FileEditTool` 操作文件时，会触发 skill 目录发现：

```
用户/模型操作 src/subproject/foo.ts
    │
    ▼
discoverSkillDirsForPaths(['src/subproject/foo.ts'], cwd)
    │
    ├── 从 src/subproject/ 开始向上遍历
    │   ├── 检查 src/subproject/.claude/skills/ → stat()
    │   ├── 检查 src/.claude/skills/ → stat()
    │   └── 到达 cwd 时停止 (cwd 级 skill 已在启动时加载)
    │
    ├── 跳过已检查的路径 (dynamicSkillDirs Set 缓存)
    ├── 跳过 gitignored 的目录
    └── 返回新发现的 skill 目录 (最深路径优先)
         │
         ▼
addSkillDirectories(dirs)
    │
    ├── projectSettings 已禁用或 plugin-only → 跳过
    ├── loadSkillsFromSkillsDir() 加载每个目录
    ├── 反序处理 (浅路径先，深路径覆盖同名)
    ├── 加入 dynamicSkills Map
    └── skillsLoaded.emit() → 通知监听器清除缓存
```

### 4.4 条件 Skill 激活

带 `paths` frontmatter 的 skill 不会立即可用，而是在文件操作匹配路径时才激活：

```typescript
// src/skills/loadSkillsDir.ts
export function activateConditionalSkillsForPaths(filePaths: string[], cwd: string): string[] {
  // 对每个待激活的条件 skill:
  //   1. 用 ignore 库 (gitignore 语法) 创建匹配器
  //   2. 将文件路径转为相对路径
  //   3. 匹配成功 → 从 conditionalSkills 移到 dynamicSkills
  //   4. 记录到 activatedConditionalSkillNames (跨缓存清除存活)
  //   5. emit 通知 → 清除命令缓存
}
```

**使用场景**: 一个测试框架相关的 skill 声明 `paths: ["**/*.test.ts", "**/*.spec.ts"]`，只有当用户操作测试文件时才会出现在可用 skill 列表中。

### 4.5 文件监视与热重载

```typescript
// src/utils/skills/skillChangeDetector.ts
// 使用 chokidar 监视 skill 目录 (深度 2)
// 文件变化 → debounced → clearSkillCaches() + clearCommandMemoizationCaches()
// 同时 resetSentSkillNames() 让下次 attachment 重新列出 skill
```

### 4.6 缓存层级

```
getSkillDirCommands()     ← memoize(cwd) → 磁盘 skill
loadAllCommands()         ← memoize(cwd) → 所有来源聚合
getSkillToolCommands()    ← memoize(cwd) → 模型可调用的 skill
getSlashCommandToolSkills() ← memoize(cwd) → 用户可调用的 skill

清除时机:
  clearSkillCaches()      → 清除磁盘 skill + 条件 skill 缓存
  clearCommandMemoizationCaches() → 清除聚合缓存 + 搜索索引
  clearCommandsCache()    → 全部清除 (命令 + 插件 + skill)
```

---

## 5. Skill 调用机制

### 5.1 两种调用路径

```
                    ┌────────────────────────────────────┐
                    │         Skill 调用                  │
                    └────────────┬───────────────────────┘
                                 │
               ┌─────────────────┼─────────────────┐
               ▼                                   ▼
    ┌──────────────────┐               ┌──────────────────┐
    │ 用户手动调用      │               │ 模型自动调用       │
    │ /skill-name args │               │ Skill Tool        │
    └────────┬─────────┘               └────────┬─────────┘
             │                                  │
             ▼                                  ▼
    processSlashCommand()              SkillTool.call()
             │                                  │
             │    ┌─────────────────────────────┘
             │    │
             ▼    ▼
    processPromptSlashCommand()
             │
      ┌──────┴──────┐
      ▼              ▼
  context=fork    context=inline
      │              │
      ▼              ▼
  子 Agent 执行   消息注入对话
```

### 5.2 Skill Tool 定义

`Skill` Tool 是模型调用 skill 的标准接口（`src/tools/SkillTool/SkillTool.ts`）：

```typescript
// 输入 schema
z.object({
  skill: z.string().describe('The skill name. E.g., "commit", "review-pr", or "pdf"'),
  args: z.string().optional().describe('Optional arguments for the skill'),
})

// 输出 schema — 两种变体
z.union([
  // Inline 模式: 消息注入
  z.object({
    success: z.boolean(),
    commandName: z.string(),
    allowedTools: z.array(z.string()).optional(),
    model: z.string().optional(),
    status: z.literal('inline'),
  }),
  // Fork 模式: 子 Agent 执行
  z.object({
    success: z.boolean(),
    commandName: z.string(),
    status: z.literal('forked'),
    agentId: z.string(),
    result: z.string(),
  }),
])
```

### 5.3 调用流程详解

#### Step 1: 验证 (`validateInput`)

```
输入: { skill: "deploy-staging", args: "main" }
  │
  ├── 空名称检查 → errorCode: 1
  ├── 去除前导 "/" (兼容性)
  ├── 远程 canonical skill 检查 (ant-only, EXPERIMENTAL_SKILL_SEARCH)
  ├── getAllCommands(context) 查找
  │   ├── 未找到 → errorCode: 2
  │   ├── disableModelInvocation → errorCode: 4
  │   └── type !== 'prompt' → errorCode: 5
  └── 验证通过 → { result: true }
```

#### Step 2: 权限检查 (`checkPermissions`)

```
权限决策流程:
  │
  ├── 1. deny 规则检查 (getRuleByContentsForTool)
  │   └── 匹配 → { behavior: 'deny' }
  │
  ├── 2. 远程 canonical skill → 自动 allow (在 deny 之后)
  │
  ├── 3. allow 规则检查
  │   ├── 精确匹配: "deploy-staging"
  │   └── 前缀匹配: "deploy:*" → 匹配 deploy-staging
  │
  ├── 4. 安全属性检查 (skillHasOnlySafeProperties)
  │   └── skill 只用了安全属性 → 自动 allow
  │       安全属性白名单: type, name, description, model, effort,
  │       source, context, agent, getPromptForCommand, ...
  │       非安全属性 (如 allowedTools, hooks) 存在时 → 需要确认
  │
  └── 5. 默认 → { behavior: 'ask', suggestions: [精确allow, 前缀allow] }
```

#### Step 3: 执行 (`call`)

**Inline 模式** (默认):

```
SkillTool.call({ skill, args })
  │
  ├── recordSkillUsage(commandName)  → 记录使用频次 (排序用)
  │
  ├── processPromptSlashCommand(commandName, args, commands, context)
  │   │
  │   ├── command.getPromptForCommand(args, toolUseContext)
  │   │   ├── 前置 "Base directory for this skill: <dir>"
  │   │   ├── substituteArguments() → 替换 $ARGUMENTS, ${arg}
  │   │   ├── 替换 ${CLAUDE_SKILL_DIR}, ${CLAUDE_SESSION_ID}
  │   │   └── executeShellCommandsInPrompt() → 执行 !`` 命令
  │   │
  │   ├── 提取 allowedTools, model, effort
  │   ├── addInvokedSkill() → 注册到状态 (compaction 保持)
  │   └── registerSkillHooks() → 注册 frontmatter 钩子
  │
  ├── tagMessagesWithToolUseID()  → 关联消息到 Tool 调用
  │
  └── 返回:
      {
        data: { success: true, commandName, allowedTools, model },
        newMessages: [...],          // 注入的用户/系统消息
        contextModifier(ctx) {       // 上下文修改器
          // 1. 合并 allowedTools 到 alwaysAllowRules
          // 2. 覆盖 mainLoopModel (保持 [1m] 后缀)
          // 3. 覆盖 effortValue
          return modifiedContext
        }
      }
```

**Fork 模式** (`context: 'fork'`):

```
executeForkedSkill(command, commandName, args, context, ...)
  │
  ├── createAgentId() → 唯一 agent ID
  │
  ├── prepareForkedCommandContext(command, args, context)
  │   ├── 构造 modifiedGetAppState (合并 allowedTools)
  │   ├── 构造 baseAgent 定义 (带 effort 覆盖)
  │   └── 构造 promptMessages (skill 内容作为系统消息)
  │
  ├── runAgent({...}) → 在隔离子 Agent 中执行
  │   ├── 独立的 token 预算
  │   ├── 独立的工具使用上下文
  │   └── 流式收集 agent 消息 → 报告进度
  │
  ├── extractResultText(agentMessages) → 提取最终结果
  │
  └── 返回:
      {
        data: { success: true, commandName, status: 'forked', agentId, result }
      }
      // 注意: Fork 没有 newMessages 和 contextModifier
      // 结果直接作为 tool_result 返回给模型
```

### 5.4 contextModifier 的作用

`contextModifier` 是 Inline 模式的关键机制——它修改后续对话的执行上下文：

```typescript
contextModifier(ctx) {
  let modifiedContext = ctx

  // 1. 合并 allowedTools — skill 声明的工具自动免确认
  if (allowedTools.length > 0) {
    modifiedContext = {
      ...modifiedContext,
      getAppState() {
        const appState = previousGetAppState()
        return {
          ...appState,
          toolPermissionContext: {
            ...appState.toolPermissionContext,
            alwaysAllowRules: {
              ...appState.toolPermissionContext.alwaysAllowRules,
              command: [...existingRules, ...allowedTools],
            },
          },
        }
      },
    }
  }

  // 2. 覆盖模型 — 保持原有的 [1m] 后缀 (上下文窗口大小)
  if (model) {
    modifiedContext = {
      ...modifiedContext,
      options: {
        ...modifiedContext.options,
        mainLoopModel: resolveSkillModelOverride(model, ctx.options.mainLoopModel),
      },
    }
  }

  // 3. 覆盖 effort 等级
  if (effort !== undefined) {
    // ... 修改 appState.effortValue
  }

  return modifiedContext
}
```

---

## 6. Skill 如何被模型"看到"

模型需要知道哪些 skill 可用。这通过三层机制实现：

### 6.1 System Prompt 中的 Skill Tool 描述

`getPrompt()` 返回 Skill Tool 的静态指令：

```
Execute a skill within the main conversation

When users ask you to perform tasks, check if any of the available skills match.
Skills provide specialized capabilities and domain knowledge.

How to invoke:
- Use this tool with the skill name and optional arguments
- Examples:
  - skill: "pdf" — invoke the pdf skill
  - skill: "commit", args: "-m 'Fix bug'" — invoke with arguments

Important:
- Available skills are listed in system-reminder messages in the conversation
- When a skill matches, invoke it BEFORE generating any other response
- NEVER mention a skill without actually calling this tool
```

### 6.2 Skill Listing Attachment (会话级注入)

每轮对话构建 attachments 时，`getSkillListingAttachments()` 会注入新发现的 skill 列表：

```
getAttachments()
  │
  ├── maybe('skill_listing', () => getSkillListingAttachments(context))
  │     │
  │     ├── getSkillToolCommands(cwd) → 过滤可模型调用的 skill
  │     │   条件: type === 'prompt'
  │     │         && !disableModelInvocation
  │     │         && source !== 'builtin'
  │     │         && (有 description 或 whenToUse)
  │     │
  │     ├── 增量检测: 只列出 sentSkillNames 中尚未发送的
  │     │
  │     └── formatCommandsWithinBudget(newSkills, contextWindowTokens)
  │           │
  │           ├── Token 预算: 上下文窗口的 1% (默认 8000 字符)
  │           ├── 每条描述硬上限: 250 字符
  │           ├── Bundled skill 描述永不截断
  │           ├── 超预算 → 按比例截断非 bundled skill 描述
  │           └── 极端超预算 → 非 bundled skill 只保留名称
  │
  └── 返回: { type: 'skill_listing', content, skillCount, isInitial }
```

### 6.3 Dynamic Skill Attachment

文件操作触发的动态 skill 通过单独的 attachment 通知模型：

```
maybe('dynamic_skill', () => getDynamicSkillAttachments(context))
  │
  ├── discoverSkillDirsForPaths(touchedPaths, cwd) → 新目录
  ├── activateConditionalSkillsForPaths(touchedPaths, cwd) → 激活条件 skill
  ├── addSkillDirectories(newDirs) → 加载 skill
  │
  └── 返回: { type: 'dynamic_skill', skillDir, skillNames, displayPath }
```

### 6.4 Token 预算控制

Skill listing 采用精细的 token 预算管理：

```typescript
// 预算 = 上下文窗口 token 数 × 4 (字符/token) × 1%
// 例: 200K 上下文 → 200000 × 4 × 0.01 = 8000 字符

const SKILL_BUDGET_CONTEXT_PERCENT = 0.01  // 1%
const CHARS_PER_TOKEN = 4
const DEFAULT_CHAR_BUDGET = 8_000
const MAX_LISTING_DESC_CHARS = 250         // 单条描述上限

// 截断策略 (formatCommandsWithinBudget):
// 1. 先用完整描述计算总量
// 2. 如果不超预算 → 原样返回
// 3. 超预算 → 分离 bundled (不截断) 和 rest
// 4. 计算每条 rest 的最大描述长度
// 5. maxDescLen < 20 → rest 只保留名称
// 6. 否则 → truncate 到 maxDescLen
```

**设计意图**: Skill 列表的作用是**发现**（让模型知道 skill 存在），不是**理解**（完整内容在调用时才加载）。所以只花 1% 上下文就够了。

---

## 7. Skill Prompt 内容处理

### 7.1 `getPromptForCommand` 处理管道

当 skill 被调用时，完整内容经过以下处理：

```
原始 SKILL.md Markdown 正文
    │
    ▼
[1] 前置 Base Directory
    "Base directory for this skill: /path/to/skill-dir\n\n" + content
    │
    ▼
[2] 参数替换 (substituteArguments)
    $ARGUMENTS → 完整参数字符串
    $branch / ${branch} → 命名参数值
    │
    ▼
[3] 目录变量替换
    ${CLAUDE_SKILL_DIR} → skill 目录绝对路径 (Windows: \ → /)
    ${CLAUDE_SESSION_ID} → 当前会话 UUID
    │
    ▼
[4] 内联 Shell 执行 (仅非 MCP skill)
    !`git status` → 替换为命令输出
    ```! ... ``` → 替换为执行结果
    │   注入 allowedTools 到 alwaysAllowRules
    │
    ▼
最终内容 → [{ type: 'text', text: finalContent }]
```

### 7.2 内联 Shell 执行

Skill 正文中可以嵌入 shell 命令，在 skill 展开时立即执行：

```markdown
当前分支信息:
!`git branch --show-current`

构建输出:
​```!
npm run build 2>&1 | tail -20
​```
```

**安全措施**:
- MCP skill (`loadedFrom === 'mcp'`) 完全禁止 shell 执行
- Shell 执行继承 skill 的 `allowedTools` 作为 `alwaysAllowRules`
- 可通过 `shell` frontmatter 指定执行 shell 类型

### 7.3 Compaction 保持

被调用的 skill 内容注册到 `addInvokedSkill()`，确保自动压缩后 skill 内容不丢失：

```
invoked_skills attachment:
  skills: [
    { name: "deploy-staging", path: "/path/to/SKILL.md", content: "处理后的完整内容" }
  ]
```

---

## 8. 用户自定义 Skill

### 8.1 创建方式

用户可以在两个层级创建 skill：

**个人级别** (跟随用户，跨所有项目):
```
~/.claude/skills/my-skill/SKILL.md
```

**项目级别** (跟随仓库，团队共享):
```
<project>/.claude/skills/my-skill/SKILL.md
```

创建后 Claude Code 会自动发现（通过 `getSkillDirCommands` 启动扫描或 `skillChangeDetector` 文件监视热重载）。

### 8.2 `/skillify` — 从会话自动生成 Skill

`/skillify` 是内置的"元技能"——分析当前会话的操作，通过多轮访谈自动生成 `SKILL.md`：

```
/skillify [描述]
  │
  ├── 分析阶段:
  │   ├── 读取 session memory summary
  │   ├── 提取所有 user messages
  │   └── 识别可重复的流程、步骤、工具、参数
  │
  ├── 访谈阶段 (4 轮 AskUserQuestion):
  │   ├── Round 1: 确认名称、描述、目标
  │   ├── Round 2: 确认步骤、参数、执行模式 (inline/fork)、保存位置
  │   ├── Round 3: 每个步骤的成功标准、人工检查点、并行性
  │   └── Round 4: 触发条件、注意事项
  │
  └── 生成阶段:
      ├── 按模板生成 SKILL.md (frontmatter + 结构化步骤)
      ├── 展示给用户 review
      └── 确认后写入文件
```

### 8.3 Skill 设计最佳实践 (从 `/skillify` 模板提炼)

```markdown
---
name: my-workflow
description: One-line description
allowed-tools:
  - "Bash(gh:*)"           # 最小权限，使用模式匹配
  - Read
when_to_use: >
  Use when the user wants to... Examples: "do X", "run Y"
arguments:
  - pr_number
argument-hint: "<PR number>"
context: fork                # 自包含任务用 fork，需要中途交互用 inline
---

# My Workflow

## Goal
明确的目标和成功标准。

## Steps

### 1. Step Name
具体操作指令。

**Success criteria**: 每步必须有！证明这步完成了。

### 2. Next Step
**Artifacts**: 这步产生什么 (PR number, commit SHA 等)
**Human checkpoint**: 不可逆操作前暂停确认
**Rules**: 硬性约束 (来自会话中用户的纠正)
```

---

## 9. 安全架构

### 9.1 权限分层

```
┌─────────────────────────────────────────────────┐
│ 编译时: feature() 消除 ant-only bundled skill    │
├─────────────────────────────────────────────────┤
│ 注册时: USER_TYPE, isEnabled() 条件检查          │
├─────────────────────────────────────────────────┤
│ 运行时: deny/allow 规则 + 安全属性自动放行       │
├─────────────────────────────────────────────────┤
│ 调用时: checkPermissions() + 用户确认弹窗        │
├─────────────────────────────────────────────────┤
│ 执行时: MCP shell 禁止 + 路径遍历保护            │
└─────────────────────────────────────────────────┘
```

### 9.2 安全属性白名单

`skillHasOnlySafeProperties()` 决定是否自动放行（不弹确认窗）：

```typescript
const SAFE_SKILL_PROPERTIES = new Set([
  // 这些属性不改变执行行为，可以自动放行:
  'type', 'name', 'description', 'model', 'effort',
  'source', 'context', 'agent', 'getPromptForCommand',
  'contentLength', 'argNames', 'pluginInfo', 'skillRoot',
  'isEnabled', 'isHidden', 'aliases', 'argumentHint',
  'whenToUse', 'version', 'disableModelInvocation',
  'userInvocable', 'loadedFrom', 'progressMessage',
  'hasUserSpecifiedDescription', 'paths', 'userFacingName',
  'immediate', 'isMcp', 'frontmatterKeys',
  'disableNonInteractive',
])

// 只要 skill 有任何不在白名单中的属性 (如 allowedTools, hooks)
// 且该属性有有意义的值 → 需要用户确认
```

**设计意图**: 新增的 PromptCommand 属性**默认需要确认**，除非显式加入白名单。这是一种 Fail-Closed 的安全策略。

### 9.3 MCP Skill 安全隔离

```
本地 Skill:
  ✓ shell 执行 (!`...`)
  ✓ ${CLAUDE_SKILL_DIR} 替换
  ✓ 文件系统访问

MCP Skill (远程):
  ✗ shell 执行 — 完全禁止
  ✗ ${CLAUDE_SKILL_DIR} — 无意义
  ✓ Markdown 正文注入 — 只有纯文本
```

### 9.4 Bundled Skill 文件提取安全

```
防御措施                     攻击面
────────────────────────    ──────────────────
per-process nonce 目录     → 防预创建目录攻击
O_NOFOLLOW | O_EXCL        → 防符号链接替换
0o700 目录 + 0o600 文件    → 防权限过宽
路径 normalize + ".." 检查  → 防路径遍历
不 unlink+retry on EEXIST  → 防 TOCTOU 竞态
```

### 9.5 Gitignore 保护

动态发现的 skill 目录会检查 gitignore 状态：

```typescript
// 防止 node_modules/malicious-pkg/.claude/skills/ 被自动加载
if (await isPathGitignored(currentDir, resolvedCwd)) {
  logForDebugging(`[skills] Skipped gitignored skills dir: ${skillDir}`)
  continue
}
```

### 9.6 Policy 控制

组织管理员可以通过 policy 限制 skill 来源：

```
isRestrictedToPluginOnly('skills')  → true 时:
  - 只加载 plugin 来源的 skill
  - 跳过 user/project 磁盘 skill
  - 跳过 legacy commands
  - 跳过动态发现
```

---

## 10. 遥测与观测

### 10.1 调用遥测

```typescript
// 每次 Skill Tool 调用:
logEvent('tengu_skill_tool_invocation', {
  command_name: sanitizedCommandName,     // 内建/bundled 用真名，自定义用 'custom'
  _PROTO_skill_name: commandName,         // PII 标记列 (不脱敏)
  execution_context: 'inline' | 'fork' | 'remote',
  invocation_trigger: 'claude-proactive' | 'nested-skill',
  query_depth: number,                    // Agent 嵌套深度
  was_discovered: boolean,                // 是否通过 skill search 发现的
  // 插件信息 (如果来源是 plugin)
  plugin_name, plugin_repository, _PROTO_plugin_name, ...
})
```

### 10.2 动态发现遥测

```typescript
// skill 动态加载或条件激活时:
logEvent('tengu_dynamic_skills_changed', {
  source: 'file_operation' | 'conditional_paths',
  previousCount, newCount, addedCount, directoryCount,
})
```

### 10.3 Skill 加载遥测

```typescript
// src/utils/telemetry/skillLoadedEvent.ts
// 每个 skill 首次加载时触发
```

### 10.4 使用频次追踪

```typescript
// src/utils/suggestions/skillUsageTracking.ts
recordSkillUsage(commandName)  // 每次调用累计
// 用于 command suggestions 排序
```

---

## 11. 架构全景

### 11.1 端到端流程

```
╔═══════════════════ 启动阶段 ═══════════════════╗
║                                                 ║
║  initBundledSkills()                            ║
║    → registerBundledSkill() × N                 ║
║    → bundledSkills[]                            ║
║                                                 ║
║  getSkillDirCommands(cwd)                       ║
║    → managed + user + project + addDir + legacy ║
║    → dedup + 分离 conditional                   ║
║    → skillDirCommands[]                         ║
║                                                 ║
║  loadAllCommands(cwd)                           ║
║    → bundled + builtin-plugin + disk            ║
║      + workflow + plugin + COMMANDS()           ║
║    → allCommands[] (memoized)                   ║
║                                                 ║
╚═════════════════════════════════════════════════╝
                      │
                      ▼
╔═══════════════ 每轮对话阶段 ════════════════════╗
║                                                 ║
║  getAttachments()                               ║
║    ├── skill_listing → 增量列出新 skill          ║
║    │   (预算 = 上下文 1%, 增量检测)              ║
║    ├── dynamic_skill → 文件操作触发的新发现       ║
║    └── skill_discovery → 实验性搜索 (gated)      ║
║                                                 ║
║  System Prompt                                  ║
║    └── Skill Tool description + 调用示例         ║
║                                                 ║
╚═════════════════════════════════════════════════╝
                      │
                      ▼
╔═══════════════════ 调用阶段 ═══════════════════╗
║                                                 ║
║  模型: Skill({ skill: "X", args: "..." })       ║
║  或用户: /X args                                 ║
║    │                                            ║
║    ├── validateInput → 存在性/类型/权限检查      ║
║    ├── checkPermissions → deny/allow/ask        ║
║    └── call()                                   ║
║        │                                        ║
║        ├── fork:                                ║
║        │   prepareForkedCommandContext()         ║
║        │   → runAgent() → 子 Agent 隔离执行      ║
║        │   → extractResultText() → 返回结果      ║
║        │                                        ║
║        └── inline:                              ║
║            processPromptSlashCommand()           ║
║            → getPromptForCommand()              ║
║              ├── Base Dir 前置                   ║
║              ├── 参数替换                        ║
║              ├── 变量替换                        ║
║              └── Shell 执行 (非 MCP)             ║
║            → newMessages + contextModifier       ║
║                                                 ║
╚═════════════════════════════════════════════════╝
                      │
                      ▼
╔═══════════════ 文件操作触发 ════════════════════╗
║                                                 ║
║  FileRead/Write/Edit 操作文件                    ║
║    │                                            ║
║    ├── discoverSkillDirsForPaths()              ║
║    │   → 向上遍历 → 发现 .claude/skills/         ║
║    │   → addSkillDirectories() → 加载到 dynamic  ║
║    │                                            ║
║    └── activateConditionalSkillsForPaths()      ║
║        → paths 模式匹配 → 从 conditional → dynamic║
║        → skillsLoaded.emit() → 清除缓存          ║
║                                                 ║
╚═════════════════════════════════════════════════╝
```

### 11.2 关键设计决策总结

| 决策 | 理由 |
|------|------|
| **Markdown + YAML frontmatter 作为定义格式** | 零学习成本、版本控制友好、不需要编译步骤 |
| **列表只暴露 name/description，内容调用时加载** | 上下文窗口资源极其宝贵，1% 预算用于发现即可 |
| **条件 skill (paths frontmatter)** | 避免无关 skill 污染模型的可用列表 |
| **动态发现 (文件操作触发)** | monorepo 场景下子项目各有 skill，按需加载 |
| **Fork vs Inline 执行模式** | 自包含任务用 fork 隔离上下文；需要交互的用 inline |
| **MCP skill 禁止 shell 执行** | 远程内容不可信，阻断 shell 注入攻击面 |
| **安全属性白名单 (Fail-Closed)** | 新属性默认需要确认，直到被审核加入白名单 |
| **realpath 去重** | 符号链接和重叠父目录不会导致同一 skill 加载两次 |
| **sentSkillNames 增量检测** | 只在 skill 池变化时注入列表，避免每轮重复浪费上下文 |
| **Bundled skill 文件提取的 nonce + O_NOFOLLOW** | 防止本地提权攻击 |

### 11.3 核心源文件索引

| 文件 | 职责 |
|------|------|
| `src/skills/loadSkillsDir.ts` | SKILL.md 解析、`createSkillCommand`、目录加载、动态/条件 skill |
| `src/skills/bundledSkills.ts` | Bundled skill 注册 API、文件安全提取 |
| `src/skills/bundled/index.ts` | `initBundledSkills()` 入口 |
| `src/skills/bundled/*.ts` | 各个内置 skill 实现 |
| `src/skills/mcpSkillBuilders.ts` | MCP skill 构建器注册 (避免循环依赖) |
| `src/commands.ts` | `getSkills`、`getCommands`、`getSkillToolCommands`、缓存管理 |
| `src/tools/SkillTool/SkillTool.ts` | Skill Tool 定义、验证、权限、执行 (inline + fork) |
| `src/tools/SkillTool/prompt.ts` | Skill Tool prompt 文本、listing 预算格式化 |
| `src/tools/SkillTool/constants.ts` | `SKILL_TOOL_NAME = 'Skill'` |
| `src/utils/attachments.ts` | `skill_listing`、`dynamic_skill`、`invoked_skills` attachment |
| `src/utils/processUserInput/processSlashCommand.tsx` | 斜杠命令 + prompt 展开 |
| `src/utils/promptShellExecution.ts` | Skill 正文中 `!` shell 命令执行 |
| `src/utils/forkedAgent.ts` | `prepareForkedCommandContext` 共享 fork 逻辑 |
| `src/utils/skills/skillChangeDetector.ts` | chokidar 文件监视 + 热重载 |
| `src/utils/hooks/registerSkillHooks.ts` | Skill frontmatter hooks 注册 |
| `src/utils/suggestions/skillUsageTracking.ts` | 使用频次追踪 |
| `src/utils/telemetry/skillLoadedEvent.ts` | 加载遥测 |
