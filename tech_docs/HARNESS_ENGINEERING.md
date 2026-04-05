---
title: Harness 工程：以 Claude Code 为例
aliases: [Harness Engineering, LLM 执行外壳]
series: Claude Code 架构解析
category: 全局架构
order: 22
tags:
  - harness
  - architecture
  - llm-orchestration
  - agent
date: 2026-04-05
---

# Harness 工程：以 Claude Code 为例

## 1. 什么是 Harness 工程

### 1.1 核心理念

**Harness**（挽具）是 AI 工程领域的一个工程理念：

> LLM 是被控制的对象，不是系统的主体。Harness 是包裹在 LLM 外面的执行外壳，负责控制模型的感知边界、行为边界和副作用边界。

类比：LLM 是"马"，harness 是套在马身上的挽具，决定马往哪走、走多快、能做什么。

### 1.2 Harness Engineer 做什么

不是 Prompt Engineer（优化怎么问模型），而是：

| 职责 | 具体内容 |
|------|---------|
| **感知边界控制** | 决定什么信息进入上下文，什么被拦截/过滤 |
| **工具执行环境** | 模型"调用"工具，harness 真正执行，并过滤结果 |
| **状态与副作用管理** | 模型无状态，harness 维护会话状态、文件系统状态 |
| **安全与权限** | 在模型动作落地之前做权限检查、用户确认 |
| **生命周期编排** | 启动、初始化、清理、多 Agent 协调 |

### 1.3 与传统软件工程的区别

```
传统工程：  用户 → 应用逻辑 → 数据库/API
Harness：   用户 → Harness → LLM → Harness → 工具执行 → Harness → 用户
```

Harness 出现在 LLM 的两侧：输入侧（构建上下文）和输出侧（处理结果、执行副作用）。

---

## 2. Claude Code 的 Harness 架构

Claude Code 是 harness 工程的完整实现，整体结构：

```
用户输入
    ↓
┌─────────────────────────────────────────┐
│              HARNESS 层                  │
│                                         │
│  ┌─────────────┐   ┌──────────────────┐ │
│  │ 输入侧       │   │ 输出侧            │ │
│  │ • Context注入 │   │ • 工具输出拦截   │ │
│  │ • Effort配置 │   │ • 权限检查       │ │
│  │ • Memory注入 │   │ • 副作用执行     │ │
│  │ • 权限前置   │   │ • Hooks触发      │ │
│  └──────┬──────┘   └────────┬─────────┘ │
└─────────┼────────────────────┼───────────┘
          ↓                    ↑
     ┌─────────┐          ┌─────────┐
     │   LLM   │ ────────→│  工具   │
     │ (无状态) │          │ 执行结果 │
     └─────────┘          └─────────┘
```

---

## 3. 具体实现：四个核心 Harness 模式

### 3.1 模式一：工具输出拦截（Side Channel）

**问题**：外部 CLI/SDK 需要向 harness 传递信息，但不能污染模型的上下文。

**解法**：`src/utils/claudeCodeHints.ts` 实现了一个旁路信道协议。

```
shell 命令执行
    ↓ 原始 stdout（含 hint 标签）
extractClaudeCodeHints()
    ├── 提取 <claude-code-hint v=1 type="plugin" value="xxx" />
    ├── 从输出中删除这些行
    └── 返回 { hints, stripped }
         ├── stripped → 送给 LLM（干净输出）
         └── hints   → harness 处理（弹出安装提示给用户）
```

关键设计：**模型永远看不到 hint 标签**，这是 harness 独占的旁路信道。

```typescript
// claudeCodeHints.ts:66
// "hints are a harness-only side channel"
export function extractClaudeCodeHints(output, command) {
  // Fast path: 无标签则零开销
  if (!output.includes('<claude-code-hint')) {
    return { hints: [], stripped: output }
  }
  // 正则提取，从输出中删除，折叠空行
  const stripped = output.replace(HINT_TAG_RE, rawLine => {
    const attrs = parseAttrs(rawLine)
    hints.push({ v, type, value, sourceCommand })
    return ''  // 从模型可见输出中抹去
  })
}
```

**可借鉴模式**：任何需要在工具输出和模型感知之间做过滤的场景，都可以用这个 "replace + side channel" 模式。

---

### 3.2 模式二：分层权限系统

**问题**：模型可能请求读写任意文件，需要在副作用发生前做多层拦截。

**解法**：`src/utils/permissions/filesystem.ts` 实现了 7 层检查流水线。

```
模型请求操作某个路径
    ↓
1. 危险文件/目录检查（.gitconfig, .ssh 等）→ 直接拒绝
2. 路径遍历攻击检测（../.. 等）           → 直接拒绝
3. 用户配置 deny 规则匹配                 → 直接拒绝
4. AutoApprove 规则匹配                   → 直接放行
5. 已授权路径（用户之前同意过）            → 直接放行
6. 工作目录内                             → 直接放行
7. Harness 内部路径（session-memory 等）  → 直接放行（harness 自己写的）
8. 用户自定义 allow 规则                  → 直接放行
    ↓ 以上都未命中
ask → 弹出权限确认对话框给用户
```

第 7 层最能体现 harness 理念：

```typescript
// filesystem.ts:1153
// "Allow reads from internal harness paths (session-memory, plans, tool-results)"
// 这些路径是 harness 自己写入的，content is harness-controlled
// 因此无需用户授权，harness 直接放行
```

**可借鉴模式**：副作用执行前的多层 Guard，越早拒绝越好（越靠前的层越轻量）。

---

### 3.3 模式三：Hooks——Harness 执行，模型不感知

**问题**：用户希望在特定事件（工具调用前后、会话开始结束等）自动触发自定义逻辑。

**解法**：`settings.json` 配置 hooks，**由 harness 执行，不是由模型执行**。

```json
// settings.json 示例
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{ "type": "command", "command": "audit-log-tool-call.sh" }]
    }],
    "PostToolUse": [{
      "hooks": [{ "type": "command", "command": "notify-slack.sh" }]
    }]
  }
}
```

执行时序：
```
模型决定调用 Bash 工具
    ↓
Harness 触发 PreToolUse hooks（模型不知道）
    ↓
Harness 真正执行 Bash 命令
    ↓
Harness 触发 PostToolUse hooks（模型不知道）
    ↓
Harness 将结果（经过过滤）返回给模型
```

**关键**：hooks 是 harness 的切面（AOP），对模型完全透明。这让用户可以在不修改 prompt 的情况下注入审计、监控、通知等逻辑。

---

### 3.4 模式四：Effort 预算控制

**问题**：不同任务需要不同深度的推理，统一用 high effort 会浪费 rate limit。

**解法**：`src/utils/effort.ts` 实现 effort 作为 harness 对模型 thinking budget 的外部控制参数。

```
用户/配置 → effortValue → Harness
                              ↓
                    resolveAppliedEffort()
                    优先级链：
                    CLAUDE_CODE_EFFORT_LEVEL 环境变量
                        → AppState (用户命令 /effort)
                            → 模型默认值
                              ↓
                    发送给 API 的 effort 参数
                    (low/medium/high/max → 控制 thinking token 预算)
```

模型能力上限不变，effort 只控制推理深度。类比：同一个人，给他 1 分钟打草稿 vs 10 分钟打草稿，答案质量不同但人的能力没变。

当前各模型默认值：

| 模型 | 默认 effort | 原因 |
|------|------------|------|
| Sonnet 4.6 | high（不传参数） | 标准工作模型 |
| Opus 4.6 + Pro/Max/Team | medium | 节省 rate limit |
| Opus 4.6 + `ultrathink` | high（临时） | 用户主动触发深度推理 |
| Opus 4.6 `max` | 仅 Opus 4.6 支持 | 最大 thinking 预算 |

---

## 4. Bootstrap：Harness 的启动管道

Harness 本身也需要初始化。`src/entrypoints/init.ts` 实现了严格有序的 4 阶段启动：

```
cli.tsx
├── Fast-path 路由（--version 零模块加载）
└── 进入 main.tsx
        ├── 检测运行模式（交互 / 无头 SDK）
        └── 调用 init()（lodash memoize，保证只运行一次）
                │
                ├── 1. 配置验证（enableConfigs）
                ├── 2. 安全环境变量（applySafeConfigEnvironmentVariables）
                │      └── CA 证书（必须在第一个 TLS 握手前完成）
                ├── 3. 优雅关闭注册（setupGracefulShutdown）
                ├── 4. 并发异步预取（Fire-and-forget）
                │      ├── 1P 事件日志初始化
                │      ├── OAuth 缓存填充
                │      ├── JetBrains IDE 检测
                │      └── Git 仓库检测
                ├── 5. 远程配置加载（initializeRemoteManagedSettingsLoadingPromise）
                ├── 6. 网络配置
                │      ├── mTLS（configureGlobalMTLS）
                │      └── HTTP 代理（configureGlobalAgents）
                ├── 7. API 预连接（preconnectAnthropicApi）
                │      └── 与步骤 4-6 并发，隐藏 TCP/TLS 握手延迟
                └── 8. 清理钩子注册（LSP、Swarm teams）
```

**设计要点**：

- **顺序约束**：CA 证书必须在第一个 TLS 握手前完成（步骤 2），mTLS/代理必须在 API 预连接前完成（步骤 6→7）
- **并发预取**：与顺序无关的初始化（OAuth、IDE 检测、Git 检测）全部 fire-and-forget 并发执行
- **防双初始化**：`memoize` 包装 + `telemetryInitialized` flag 双重保障
- **懒加载**：OpenTelemetry（~400KB）、gRPC（~700KB）在真正需要时才动态 import

---

## 5. Harness 的状态管理

`src/bootstrap/state.ts` 是 harness 的全局状态容器，覆盖整个进程生命周期：

```typescript
type State = {
  originalCwd: string       // 启动时的工作目录（不随 cd 变化）
  projectRoot: string       // 稳定的项目根（用于 history/skills/sessions 身份）
  totalCostUSD: number      // 累计 API 费用
  totalAPIDuration: number  // 累计 API 耗时
  modelUsage: {...}         // 按模型分类的 token 用量
  sessionId: SessionId      // 唯一会话 ID
  isInteractive: boolean    // 是否交互模式
  kairosActive: boolean     // KAIROS 主动模式是否激活
  // ... 30+ 字段
}
```

注释 `// DO NOT ADD MORE STATE HERE - BE JUDICIOUS WITH GLOBAL STATE` 体现了对全局状态的克制原则——harness 只在 bootstrap 层维护真正全局的状态，其余状态下推到各模块。

---

## 6. Harness 控制 LLM 的三个维度

总结 Claude Code 中 harness 控制 LLM 的三个维度：

### 维度一：感知边界（什么进入上下文）

```
原始数据源
├── 用户输入
├── 项目文件（CLAUDE.md、.claude/ 目录）
├── Memory（session-memory、auto-mem）
├── 工具执行结果（经过 extractClaudeCodeHints 过滤）
└── System Prompt（harness 动态构建）
        ↓ 全部由 harness 组装
    模型看到的上下文（harness 决定边界）
```

### 维度二：行为边界（模型能做什么）

```
模型输出工具调用请求
        ↓
Harness 权限检查（7层）
        ↓
Harness 用户确认（如需要）
        ↓
Harness 真正执行工具
        ↓
Harness 过滤结果再返回给模型
```

### 维度三：推理深度（模型思考多深）

```
任务类型 → Effort 配置 → API effort 参数 → thinking token 预算
```

---

## 7. 构建自己的 Harness：关键设计决策

基于 Claude Code 的实践，构建 LLM harness 时需要决策的核心问题：

### 7.1 状态归属

| 状态类型 | 归属 | 理由 |
|---------|------|------|
| 会话 ID、累计费用 | Harness | 跨工具调用持久 |
| 当前工作目录 | Harness | 模型不应直接控制 |
| 对话历史 | Harness | 模型每次调用都是无状态的 |
| 工具执行结果 | Harness 暂存 | 需要过滤后再送给模型 |

### 7.2 拦截点设计

```
输入拦截：  用户消息 → [注入 context] → 模型
输出拦截：  模型输出 → [解析工具调用] → [权限检查] → 执行
结果拦截：  工具结果 → [过滤 side channel] → [spilling 处理] → 模型
```

### 7.3 副作用隔离原则

- **可逆性**：优先执行可撤销的操作（文件修改可 git revert，危险命令要确认）
- **最小权限**：默认拒绝，显式授权才放行
- **审计可追溯**：所有工具调用记录到 session log

### 7.4 Effort / 推理预算

不同任务给不同思考预算，而不是统一用最高档：

```
简单格式转换    → low
日常编码        → medium
复杂架构设计    → high
需要深度推理    → max（仅限支持的模型）
```

---

## 8. 与其他概念的关系

| 概念 | 关系 |
|------|------|
| **Prompt Engineering** | Harness 包含 prompt 构建，但远不止于此 |
| **RAG** | Harness 的 context 注入可以包含 RAG，是其一个子组件 |
| **Agent Framework** | Harness 是 agent framework 的底层执行环境 |
| **MCP** | MCP 是 harness 扩展工具集的协议标准 |
| **LangChain/LlamaIndex** | 这些框架本质上是 harness 的实现，Claude Code 是另一种实现 |

---

## 9. 小结

Claude Code 作为 harness 工程的完整实现，核心贡献在于：

1. **工具输出旁路信道**：`extractClaudeCodeHints` 在模型感知之前做拦截，实现 harness-only 的元信息传递
2. **7 层权限流水线**：副作用发生前的多级防御，越早拒绝越轻量
3. **Hooks 切面系统**：对模型透明的生命周期事件，用于审计、监控、自动化
4. **Effort 外部控制**：harness 统一管理 LLM 的推理预算，不依赖 prompt 技巧
5. **有序启动管道**：严格的初始化顺序 + 并发预取，在安全约束内最大化启动速度

这套模式的本质是：**把 LLM 从"应用主体"降级为"推理组件"**，所有状态、权限、副作用、生命周期都由 harness 统一管理。
