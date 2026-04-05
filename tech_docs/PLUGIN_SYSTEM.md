---
title: Plugin 系统
aliases: [Plugin 系统, Plugin System]
series: Claude Code 架构解析
category: 扩展生态
order: 14
tags:
  - claude-code
  - plugins
  - marketplace
  - enterprise
  - hot-reload
date: 2026-04-04
---

# Plugin 系统详细技术文档

## 1. 概述

Claude Code 的 Plugin 系统是一个**基于 Marketplace 的扩展生态**，允许通过插件向 Claude Code 添加斜杠命令、Skills、Agents、Hooks、MCP 服务器和 LSP 服务器。插件通过 Git 仓库形式的 Marketplace 分发，支持企业策略管控、自动更新和版本锁定。

### 核心能力

- **组件扩展**：Commands / Skills / Agents / Hooks / MCP Servers / LSP Servers / Output Styles
- **多源管理**：Marketplace (Git/GitHub/npm/pip/URL) + Built-in + Inline (--plugin-dir)
- **企业管控**：Marketplace 白名单/黑名单、策略锁定、Managed-only 模式
- **安全防护**：路径遍历防护、Manifest 校验、名称冒充检测、Agent 权限限制
- **运行时管理**：热重载、SDK 刷新、自动更新、依赖解析

---

## 2. 架构总览

```
┌─────────────────────────────────────────────────────────────────┐
│                          Plugin 来源                              │
│                                                                   │
│  Marketplace (Git 仓库)      --plugin-dir      Built-in          │
│  ┌──────────────────┐      ┌──────────────┐  ┌──────────────┐   │
│  │marketplace.json  │      │ name@inline  │  │ name@builtin │   │
│  │  → plugins[]     │      │ 本地路径     │  │ 内存注册     │   │
│  │    → source      │      └──────────────┘  └──────────────┘   │
│  │    → plugin.json │                                            │
│  └──────────────────┘                                            │
└──────────────┬──────────────────┬──────────────────┬─────────────┘
               │                  │                  │
               ▼                  ▼                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                    pluginLoader.ts                                │
│                                                                   │
│  loadAllPlugins() / loadAllPluginsCacheOnly()                    │
│  assemblePluginLoadResult():                                     │
│    1. loadPluginsFromMarketplaces() ← 策略过滤                   │
│    2. loadSessionOnlyPlugins() ← --plugin-dir                    │
│    3. getBuiltinPlugins() ← 内置                                 │
│    4. mergePluginSources() ← 合并 + 去重 (session > marketplace) │
│    5. verifyAndDemote() ← 依赖检查                               │
│    6. cachePluginSettings() ← 插件设置注入                       │
│                                                                   │
│  结果: PluginLoadResult { enabled[], disabled[], errors[] }      │
└──────────────────────────┬──────────────────────────────────────┘
                           │
           ┌───────────────┼───────────────┬──────────────┐
           ▼               ▼               ▼              ▼
  loadPluginCommands  loadPluginAgents  loadPluginHooks  MCP/LSP
  getPluginSkills     loadPluginAgents  registerHook     getPlugin
                                        Callbacks()      McpServers
           │               │               │              │
           ▼               ▼               ▼              ▼
    commands.ts       loadAgentsDir.ts  bootstrap/     mcp/config.ts
    loadAllCommands   getAgentDefs...   state.ts       lsp/config.ts
```

---

## 3. Plugin 目录结构

### 3.1 标准 Plugin 布局

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json          ← Plugin Manifest (必需)
├── commands/                ← 斜杠命令 (Markdown 文件)
│   ├── format.md
│   └── lint.md
├── skills/                  ← Skills (含 SKILL.md 的目录)
│   └── code-review/
│       └── SKILL.md
├── agents/                  ← Agent 定义 (Markdown 文件)
│   └── reviewer.md
├── hooks/
│   └── hooks.json           ← Hook 配置
├── output-styles/           ← 输出风格
│   └── minimal.md
├── .mcp.json                ← MCP 服务器配置
├── .lsp.json                ← LSP 服务器配置
└── settings.json            ← 插件贡献的设置 (受限 key)
```

### 3.2 Marketplace 布局

```
my-marketplace/
├── .claude-plugin/
│   └── marketplace.json     ← Marketplace Manifest
└── plugins/
    ├── formatter/           ← 插件目录 (相对路径)
    │   └── .claude-plugin/
    │       └── plugin.json
    └── linter/
        └── ...
```

---

## 4. Manifest 格式

### 4.1 Plugin Manifest (`plugin.json`)

```json
{
  "name": "my-formatter",
  "version": "1.2.0",
  "description": "Code formatting tools for Claude Code",
  "author": {
    "name": "Example Corp",
    "email": "dev@example.com",
    "url": "https://github.com/example"
  },
  "homepage": "https://github.com/example/formatter",
  "license": "MIT",
  "keywords": ["formatting", "code-quality"],
  "dependencies": ["linter@my-marketplace"],

  "commands": ["./extra-commands/deploy.md"],
  "agents": ["./custom-agents/qa.md"],
  "skills": ["./custom-skills/"],
  "hooks": { "PreToolUse": [{ "matcher": "Write", "hooks": [...] }] },

  "mcpServers": {
    "formatter-server": {
      "command": "node",
      "args": ["${CLAUDE_PLUGIN_ROOT}/server.js"]
    }
  },
  "lspServers": {
    "ts-lint": {
      "command": "node",
      "args": ["./lsp-server.js", "--stdio"]
    }
  },

  "channels": {
    "formatter-channel": {
      "mcpServer": "formatter-server",
      "description": "Formatting notifications"
    }
  },

  "userConfig": {
    "indentSize": {
      "type": "number",
      "description": "Number of spaces for indentation",
      "default": 2
    },
    "style": {
      "type": "string",
      "description": "Formatting style",
      "enum": ["prettier", "standard"]
    }
  },

  "settings": {
    "agent": { ... }
  }
}
```

**Manifest Schema 关键字段**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | string | 唯一标识符 (kebab-case, 无空格) |
| `version` | string? | Semver 版本号 |
| `description` | string? | 用户可见描述 |
| `author` | object? | 作者信息 (name, email, url) |
| `dependencies` | string[]? | 依赖的其他插件 (bare name → 同 marketplace) |
| `commands` | path\|path[]\|object | 额外命令文件/目录 |
| `agents` | path\|path[] | 额外 Agent 定义 |
| `skills` | path\|path[] | 额外 Skill 目录 |
| `hooks` | path\|HooksSettings | Hook 配置 (JSON 路径或内联) |
| `mcpServers` | Record | MCP 服务器配置 |
| `lspServers` | Record | LSP 服务器配置 |
| `channels` | Record | Channel 配置 (助手模式推送) |
| `userConfig` | Record | 用户可配置选项 |
| `settings` | object | 贡献到全局设置 (受限 allowlist) |

### 4.2 Marketplace Manifest (`marketplace.json`)

```json
{
  "name": "my-marketplace",
  "description": "A curated collection of tools",
  "owner": "Example Corp",
  "plugins": [
    {
      "name": "formatter",
      "source": "./plugins/formatter",
      "description": "Code formatter",
      "strict": true
    },
    {
      "name": "remote-tool",
      "source": {
        "source": "github",
        "repo": "example/remote-tool",
        "path": "plugin"
      }
    }
  ]
}
```

### 4.3 Plugin Source 类型

Marketplace 中的插件可以来自多种来源：

| Source 类型 | 格式 | 说明 |
|------------|------|------|
| 相对路径 | `"./plugins/foo"` | Marketplace 仓库内的目录 |
| GitHub | `{ source: "github", repo: "owner/repo" }` | GitHub 仓库 |
| Git | `{ source: "git", url: "..." }` | 任意 Git URL |
| Git Subdir | `{ source: "git-subdir", url: "...", path: "..." }` | Git 仓库的子目录 |
| npm | `{ source: "npm", package: "..." }` | npm 包 |
| pip | `{ source: "pip", package: "..." }` | Python pip 包 |
| URL | `{ source: "url", url: "..." }` | 下载 URL |
| Settings | `{ source: "settings", name: "..." }` | 内联合成 |

---

## 5. Plugin 标识体系

### 5.1 Plugin ID 格式

```
{plugin-name}@{marketplace-name}
```

示例：
- `formatter@my-marketplace` — marketplace 中的插件
- `my-tool@inline` — `--plugin-dir` 加载的会话插件
- `voice@builtin` — 内置插件

### 5.2 标识符解析

`pluginIdentifier.ts` 提供核心解析：

```typescript
parsePluginIdentifier('formatter@my-marketplace')
// → { name: 'formatter', marketplace: 'my-marketplace' }

buildPluginId('formatter', 'my-marketplace')
// → 'formatter@my-marketplace'
```

### 5.3 保留名称

- `inline` — `--plugin-dir` 会话插件专用
- `builtin` — 内置插件专用
- 官方 Marketplace 名称 (`claude-plugins-official`, `anthropic-marketplace` 等) — 仅允许来自 `anthropics` GitHub 组织

---

## 6. 加载生命周期

### 6.1 完整加载流程

```
1. 声明 (Settings)
   enabledPlugins: { "formatter@my-marketplace": true }
   extraKnownMarketplaces: [{ source: "github", repo: "..." }]
         │
         ▼
2. 调和 (Reconcile)
   reconcileMarketplaces()
   - 合并 implicit official + explicit extra
   - 写入/更新 known_marketplaces.json
   - 同步 marketplace Git 仓库到缓存
         │
         ▼
3. 加载 (Load)
   loadAllPlugins() / loadAllPluginsCacheOnly()
         │
         ├── loadPluginsFromMarketplaces()
         │     ├── getDeclaredMarketplaces()
         │     ├── 遍历 enabledPlugins
         │     ├── getPluginById() → marketplace entry
         │     ├── isSourceAllowedByPolicy() ← 企业策略
         │     ├── createPluginFromPath() → LoadedPlugin
         │     │     ├── loadPluginManifest()
         │     │     ├── 加载 hooks/hooks.json
         │     │     ├── 合并 manifest hooks
         │     │     └── 解析 commands/agents/skills 路径
         │     └── cachePlugin() (版本化缓存)
         │
         ├── loadSessionOnlyPlugins() ← --plugin-dir
         ├── getBuiltinPlugins() ← 内置注册表
         │
         ├── mergePluginSources()
         │     └── session > marketplace (按名去重)
         │
         ├── verifyAndDemote() ← 依赖检查
         └── cachePluginSettings() ← 设置注入
         │
         ▼
4. 注册 (Register)
   ┌─ getPluginCommands() → commands.ts
   ├─ getPluginSkills() → commands.ts (via getSkills)
   ├─ loadPluginAgents() → loadAgentsDir.ts
   ├─ loadPluginHooks() → registerHookCallbacks()
   ├─ getPluginMcpServers() → mcp/config.ts
   ├─ loadPluginLspServers() → lsp/config.ts
   └─ loadPluginOutputStyles() → output styles
```

### 6.2 Cache-Only 模式

`loadAllPluginsCacheOnly()` 是默认的加载模式——它从本地缓存读取已安装的插件，不触发网络请求。只有在以下情况下才使用 `loadAllPlugins()`（可能触发网络）：

- 首次安装
- 用户手动刷新 (`/plugins`)
- `CLAUDE_CODE_SYNC_PLUGIN_INSTALL` 环境变量

### 6.3 版本化缓存

```
~/.claude/plugins/
├── marketplaces/
│   └── my-marketplace/         ← marketplace Git clone
│       └── marketplace.json
├── plugins/
│   └── formatter/
│       └── 1.2.0/             ← 版本化缓存目录
│           └── .claude-plugin/
│               └── plugin.json
└── installed_plugins.json      ← 安装元数据
```

`pluginVersioning.ts` 计算版本：semver > git SHA > `unknown`。

---

## 7. 组件集成

### 7.1 Commands / Skills

```typescript
// loadPluginCommands.ts
export const getPluginCommands = memoize(async () => {
  const { enabled } = await loadAllPluginsCacheOnly()
  // 遍历每个 plugin 的 commandsPath / commandsPaths / commandsMetadata
  // Markdown → Command 对象, source: 'plugin'
})

export const getPluginSkills = memoize(async () => {
  // 遍历 skillsPath / skillsPaths
  // SKILL.md → Command 对象, source: 'plugin'
})
```

**命名空间**：命令注册后以 `/plugin-name:command-name` 形式可用（object-mapping 格式）或直接以文件名作为命令名。

**集成到 commands.ts**：

```typescript
const loadAllCommands = memoize(async (cwd) => [
  ...bundledSkills,
  ...builtinPluginSkills,
  ...skillDirCommands,
  ...workflowCommands,
  ...pluginCommands,     // ← 插件命令
  ...pluginSkills,       // ← 插件 skills
  ...COMMANDS(),
])
```

### 7.2 Agents

```typescript
// loadPluginAgents.ts
export const loadPluginAgents = memoize(async () => {
  const { enabled } = await loadAllPluginsCacheOnly()
  // 遍历 agentsPath / agentsPaths
  // Markdown → AgentDefinition, source: 'plugin'
  // 安全限制: 忽略 permissionMode, hooks, mcpServers 字段
})
```

**安全限制**：Plugin Agent 不能通过 frontmatter 设置 `permissionMode`、`hooks` 或 `mcpServers`——这些字段只在项目级 `.claude/agents/` 中有效。

### 7.3 Hooks

```typescript
// loadPluginHooks.ts
export function loadPluginHooks(plugins: LoadedPlugin[]) {
  // 遍历 plugin.hooksConfig
  // 转换为 PluginHookMatcher[] (含 pluginRoot, pluginId, pluginName)
  clearRegisteredPluginHooks()
  registerHookCallbacks(allPluginHooks)  // → bootstrap/state.ts
}
```

Plugin Hooks 通过 `registerHookCallbacks()` 注入到全局 Hook 注册表，与 settings.json 中的 hooks 一起参与 `getHooksConfig()` 合并。

### 7.4 MCP 服务器

```typescript
// mcpPluginIntegration.ts
export async function getPluginMcpServers(plugin: LoadedPlugin) {
  // 来源 1: .mcp.json 文件
  // 来源 2: manifest mcpServers 字段
  // 来源 3: MCPB (.mcpb / .dxt) 文件
  //
  // 环境变量替换:
  //   ${CLAUDE_PLUGIN_ROOT} → 插件根目录
  //   ${CLAUDE_PLUGIN_DATA} → 插件数据目录
  //   ${user_config.key} → pluginConfigs 中的用户配置值
  //
  // 返回: Record<string, ScopedMcpServerConfig>
}
```

Plugin MCP 服务器在 `mcp/config.ts` 中与其他 MCP 来源合并，以 `scope: 'plugin'` 标识。

### 7.5 LSP 服务器

```typescript
// lspPluginIntegration.ts
export async function loadPluginLspServers(plugin: LoadedPlugin) {
  // 来源: .lsp.json + manifest lspServers
  // 路径安全: validatePathWithinPlugin() 确保命令在插件目录内
}
```

### 7.6 Settings 贡献

Plugin 可以通过 `settings.json` 贡献设置，但仅限于 `PluginSettingsSchema` 定义的 **allowlisted keys**：

```typescript
// 当前仅允许 agent 相关设置
const PluginSettingsSchema = z.object({
  agent: z.object({ ... }).optional()
})
```

插件设置在 `cachePluginSettings()` 中合并为全局设置的**最低优先级基础层**。

---

## 8. 三种 Plugin 来源

### 8.1 Marketplace Plugin

主要分发方式。通过 Git 仓库形式的 Marketplace 管理：

```json
// settings.json
{
  "enabledPlugins": {
    "formatter@my-marketplace": true,
    "linter@my-marketplace": true
  },
  "extraKnownMarketplaces": [
    { "source": "github", "repo": "example/my-marketplace" }
  ]
}
```

### 8.2 Inline Plugin (`--plugin-dir`)

开发和调试用——直接指定本地目录：

```bash
claude --plugin-dir /path/to/my-plugin
```

标识为 `my-plugin@inline`，优先级高于同名 marketplace 插件。

### 8.3 Built-in Plugin

随 CLI 发行的内置插件，通过 `registerBuiltinPlugin()` 在内存中注册：

```typescript
// builtinPlugins.ts
registerBuiltinPlugin({
  name: 'voice',
  description: 'Voice input support',
  skills: [...],
  hooks: { ... },
  mcpServers: { ... },
  defaultEnabled: true,
  isAvailable: () => checkSystemCapability()
})
```

标识为 `voice@builtin`，在 `/plugins` UI 中可启用/禁用，但不可更新/卸载。

---

## 9. Marketplace 管理

### 9.1 Marketplace 声明与调和

```typescript
// reconciler.ts
function reconcileMarketplaces(
  declared: Map<string, MarketplaceSource>,
  known: KnownMarketplacesFile
): MarketplaceReconcileResult
```

调和流程确保 `known_marketplaces.json`（磁盘状态）与声明的 marketplace 来源一致：
- 新增：声明但不在 known 中 → 添加
- 移除：在 known 中但不再声明 → 标记移除
- 更新：来源变更 → 更新

### 9.2 Official Marketplace

```typescript
// officialMarketplace.ts
export const OFFICIAL_MARKETPLACE_NAME = 'claude-plugins-official'
export const OFFICIAL_MARKETPLACE_SOURCE = {
  source: 'github', repo: 'anthropics/claude-plugins-official'
}
```

官方 Marketplace 在用户首次启用 `*@claude-plugins-official` 插件时隐式注册。也支持 GCS 备用获取路径。

### 9.3 自动更新

```typescript
// pluginAutoupdate.ts
export async function autoUpdateMarketplacesAndPluginsInBackground()
```

启动时在后台执行：
1. 刷新所有 `autoUpdate: true` 的 marketplace
2. 检查插件版本更新
3. 更新 `installed_plugins.json`

官方 marketplace 默认 `autoUpdate: true`（除 `knowledge-work-plugins`）。

---

## 10. 安全机制

### 10.1 名称冒充防护

```typescript
// schemas.ts
const BLOCKED_OFFICIAL_NAME_PATTERN = 
  /(?:official.*(?:anthropic|claude)|(?:anthropic|claude).*official|^(?:anthropic|claude).*(?:marketplace|plugins|official))/i

// 同形字攻击防护
const NON_ASCII_PATTERN = /[^\u0020-\u007E]/

// 保留名称来源校验
function validateOfficialNameSource(name, source): string | null {
  // 保留名称只能来自 anthropics GitHub 组织
}
```

### 10.2 路径安全

```typescript
// pluginInstallationHelpers.ts
function validatePathWithinBase(resolvedPath, baseDir): void {
  // 防止路径遍历 (../../etc/passwd)
  if (!resolvedPath.startsWith(baseDir)) throw Error
}

// lspPluginIntegration.ts
function validatePathWithinPlugin(path, pluginRoot): void {
  // LSP 命令必须在插件目录内
}
```

### 10.3 Agent 权限限制

`loadPluginAgents.ts` 明确忽略插件 Agent 中的安全敏感字段：

```typescript
// 插件 Agent 不能设置:
// - permissionMode (避免提权)
// - hooks (避免注入恶意 hooks)
// - mcpServers (避免注入恶意 MCP)
// 这些字段仅在项目级 .claude/agents/ 中有效
```

### 10.4 HTTP Hook 安全

插件提供的 HTTP Hooks 受 `allowedHttpHookUrls` 白名单和 SSRF 防护约束。

### 10.5 MCP 服务器去重

当多个插件声明相同的 MCP 服务器（相同 command/URL）时，后者被抑制并记录为 `mcp-server-suppressed-duplicate` 错误。

---

## 11. 企业策略

### 11.1 策略设置

| 设置项 | 说明 |
|--------|------|
| `strictKnownMarketplaces` | Marketplace 来源白名单 (仅允许列出的来源) |
| `blockedMarketplaces` | Marketplace 来源黑名单 |
| `strictPluginOnlyCustomization` | 锁定为仅插件自定义 |
| `enabledPlugins` (managed) | 管理员强制启用/禁用的插件 |
| `channelsEnabled` | 是否启用 Channel 功能 |
| `allowedChannelPlugins` | Channel 插件白名单 |
| `pluginTrustMessage` | 自定义信任提示消息 |

### 11.2 策略过滤

```typescript
// marketplaceHelpers.ts
function isSourceAllowedByPolicy(source: MarketplaceSource): boolean {
  // 1. 检查 blockedMarketplaces (黑名单)
  // 2. 检查 strictKnownMarketplaces (白名单)
  // 3. 支持 hostPattern / pathPattern 模式匹配
}
```

策略在加载阶段生效——不在白名单中的 marketplace 的插件会被标记为 `marketplace-blocked-by-policy` 错误并跳过。

---

## 12. User Config (用户配置)

插件可以声明用户可配置的选项，存储在 settings.json 的 `pluginConfigs` 中：

```json
// settings.json
{
  "pluginConfigs": {
    "formatter@my-marketplace": {
      "mcpServers": {
        "formatter-server": {
          "PORT": "3000"
        }
      },
      "options": {
        "indentSize": 4,
        "style": "prettier"
      }
    }
  }
}
```

**变量替换**：MCP 服务器配置中的 `${user_config.key}` 会被替换为用户配置的值。

```json
// plugin.json 中的 MCP 配置
{
  "mcpServers": {
    "my-server": {
      "command": "my-server",
      "args": ["--port", "${user_config.port}"],
      "env": {
        "API_KEY": "${user_config.apiKey}"
      }
    }
  }
}
```

---

## 13. MCPB (MCP Bundle)

MCPB 是一种打包格式，将 MCP 服务器及其运行时依赖封装为单个可分发文件：

```typescript
// mcpbHandler.ts
async function loadMcpbFile(path: string): Promise<McpServerConfig> {
  // 支持 .mcpb 和 .dxt 扩展名
  // 支持本地路径和远程 URL
  // 提取后验证 manifest + user config
}
```

插件在 manifest 中引用 MCPB：

```json
{
  "mcpServers": {
    "bundled-tool": "./tools/my-tool.mcpb"
  }
}
```

---

## 14. 依赖系统

### 14.1 声明依赖

```json
// plugin.json
{
  "dependencies": [
    "linter@my-marketplace",
    "utils"
  ]
}
```

Bare name (`"utils"`) 自动解析为同一 marketplace 的插件。

### 14.2 依赖解析

```typescript
// dependencyResolver.ts
function verifyAndDemote(
  plugins: LoadedPlugin[],
  allPlugins: LoadedPlugin[]
): { demoted: LoadedPlugin[], errors: PluginError[] }
```

- 依赖未启用 → `dependency-unsatisfied (not-enabled)` 错误，依赖者降级为 disabled
- 依赖不存在 → `dependency-unsatisfied (not-found)` 错误

---

## 15. Headless 安装

```typescript
// headlessPluginInstall.ts
export async function installPluginsForHeadless(): Promise<boolean> {
  // 1. registerSeedMarketplaces() — 注册种子 marketplace
  // 2. reconcileMarketplaces() — 调和 (可选跳过非 zip 兼容源)
  // 3. syncMarketplacesToZipCache() — 同步到 zip 缓存
  // 4. detectAndUninstallDelistedPlugins() — 清理已下架插件
  // 5. 返回 pluginsChanged (通知调用者刷新)
}
```

适用于 CI/CD、容器化部署等无交互环境。Zip 缓存模式避免在临时文件系统上执行 Git clone。

---

## 16. 热重载与 SDK 刷新

### 16.1 热重载

```typescript
// refresh.ts
export async function refreshActivePlugins(setAppState) {
  clearPluginCache()           // 清除 memoize 缓存
  const result = await loadAllPlugins()
  // 更新 AppState.plugins (commands, agents, hooks, mcp, lsp, outputStyles)
  // 重新注册 hooks
  // 触发 MCP 重连
}
```

### 16.2 SDK Control

SDK 消费者通过 `reload_plugins` 控制请求触发刷新：

```json
{ "type": "control_request", "request_id": "...",
  "request": { "subtype": "reload_plugins" } }
```

响应包含刷新后的 commands、agents、plugins、mcpServers 列表。

### 16.3 Settings 变更触发

`setupPluginHookHotReload` 监听设置变更，当 `enabledPlugins` / `pluginConfigs` 等变化时自动重新加载 hooks。

---

## 17. 错误处理

`PluginError` 是一个 **discriminated union** (24+ 种错误类型)：

| 错误类型 | 说明 |
|----------|------|
| `generic-error` | 通用加载错误 |
| `plugin-not-found` | 插件不在 marketplace 中 |
| `plugin-cache-miss` | 插件未缓存 |
| `path-not-found` | 组件路径不存在 |
| `manifest-parse-error` | Manifest JSON 解析失败 |
| `manifest-validation-error` | Manifest 校验失败 |
| `git-auth-failed` | Git 认证失败 (SSH/HTTPS) |
| `git-timeout` | Git clone/pull 超时 |
| `network-error` | 网络请求失败 |
| `marketplace-not-found` | Marketplace 不存在 |
| `marketplace-load-failed` | Marketplace 加载失败 |
| `marketplace-blocked-by-policy` | 被企业策略阻止 |
| `mcp-config-invalid` | MCP 配置无效 |
| `mcp-server-suppressed-duplicate` | MCP 服务器重复 |
| `hook-load-failed` | Hook 加载失败 |
| `component-load-failed` | 组件加载失败 |
| `dependency-unsatisfied` | 依赖未满足 |
| `mcpb-*` | MCPB 下载/提取/校验失败 |
| `lsp-*` | LSP 配置/启动/崩溃/超时/请求失败 |

每种错误类型携带特定的上下文数据，用于精确的 UI 展示和调试。

---

## 18. 关键文件索引

| 文件 | 角色 |
|------|------|
| `src/utils/plugins/pluginLoader.ts` | 核心加载引擎 |
| `src/utils/plugins/schemas.ts` | Zod Manifest/Marketplace schemas |
| `src/types/plugin.ts` | TypeScript 类型 (LoadedPlugin, PluginError) |
| `src/utils/plugins/marketplaceManager.ts` | Marketplace 管理 (fetch/cache/CRUD) |
| `src/utils/plugins/marketplaceHelpers.ts` | 企业策略过滤 |
| `src/utils/plugins/reconciler.ts` | Marketplace 调和 |
| `src/utils/plugins/installedPluginsManager.ts` | 安装元数据管理 |
| `src/utils/plugins/pluginIdentifier.ts` | ID 解析 (name@marketplace) |
| `src/utils/plugins/loadPluginCommands.ts` | 命令/Skill 加载 |
| `src/utils/plugins/loadPluginAgents.ts` | Agent 加载 (含安全限制) |
| `src/utils/plugins/loadPluginHooks.ts` | Hook 加载与注册 |
| `src/utils/plugins/mcpPluginIntegration.ts` | MCP 服务器集成 |
| `src/utils/plugins/lspPluginIntegration.ts` | LSP 服务器集成 |
| `src/utils/plugins/pluginOptionsStorage.ts` | 用户配置存储与变量替换 |
| `src/utils/plugins/headlessPluginInstall.ts` | Headless 安装 |
| `src/utils/plugins/pluginAutoupdate.ts` | 自动更新 |
| `src/utils/plugins/refresh.ts` | 热重载编排 |
| `src/utils/plugins/dependencyResolver.ts` | 依赖解析 |
| `src/plugins/builtinPlugins.ts` | 内置插件注册表 |
| `src/utils/settings/types.ts` | 设置中的插件配置项 |

---

## 19. 关键设计决策

### 19.1 Marketplace 模型 (非注册表)

**决策：** 使用 Git 仓库作为 Marketplace，而非中心化注册表 (如 npm registry)。

**原因：** Git 仓库天然支持版本控制、审计追踪、分叉和企业自托管。企业可以运行自己的 Marketplace 仓库，无需依赖外部服务。

### 19.2 Cache-Only 默认

**决策：** 默认使用 `loadAllPluginsCacheOnly()`，不触发网络。

**原因：** 启动性能。插件加载发生在关键路径上，网络请求会显著增加启动延迟。更新在后台异步执行。

### 19.3 Plugin Agent 安全限制

**决策：** 插件 Agent 不能设置 `permissionMode`、`hooks`、`mcpServers`。

**原因：** 防止第三方插件通过 Agent 提权。这些敏感字段只在用户/项目级别可信的 `.claude/agents/` 中有效。

### 19.4 Settings Allowlist

**决策：** 插件只能贡献 `PluginSettingsSchema` 中定义的设置项。

**原因：** 防止插件覆盖安全关键设置（如权限模式、API key 等）。当前 allowlist 非常窄（仅 `agent` 相关），可按需扩展。

### 19.5 Discriminated Union 错误类型

**决策：** 使用 24+ 种 discriminated union 错误类型替代字符串错误。

**原因：** 每种错误携带特定上下文数据（路径、URL、marketplace 名等），支持精确的 UI 展示、故障排查和自动化处理。避免了字符串匹配的脆弱性。
