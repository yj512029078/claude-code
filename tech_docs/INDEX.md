---
title: Claude Code 技术文档索引
aliases: [索引, MOC, 目录]
series: Claude Code 架构解析
tags:
  - claude-code
  - MOC
  - index
date: 2026-04-04
cssclass: wide-page
---

# Claude Code 技术文档索引

> 本项目共 21 篇技术文档，约 18,500 行，覆盖 Claude Code 的完整架构。以下按**架构层次**组织，从全局视角到具体模块，由核心到外围。

---

## 一、全局架构

从整体视角理解项目——架构蓝图、设计哲学、安全基石。

| 文档 | 说明 | 规模 |
|------|------|------|
| [[ARCHITECTURE_OVERVIEW\|架构概览]] | 项目结构、核心组件关系、启动流程、技术栈 (TypeScript/React-Ink/Commander/Bun) | ~680 行 |
| [[DESIGN_HIGHLIGHTS\|设计亮点]] | 可借鉴的工程模式：启动优化、并发预取、上下文管理、动态加载 | ~610 行 |
| [[SECURITY_ARCHITECTURE\|安全架构]] | 7 层纵深防御：权限模式、声明式规则、BashTool AST 分析、沙箱、信息安全 | ~620 行 |
| [[HARNESS_ENGINEERING\|Harness 工程]] | LLM 执行外壳理念：工具拦截、权限流水线、Hooks 切面、Effort 预算控制、Bootstrap 管道 | ~280 行 |

**阅读建议**：先读 [[ARCHITECTURE_OVERVIEW\|架构概览]] 建立全貌，再读 [[DESIGN_HIGHLIGHTS\|设计亮点]] 理解设计思路，[[HARNESS_ENGINEERING\|Harness 工程]] 理解 LLM 编排理念，[[SECURITY_ARCHITECTURE\|安全架构]] 可按需深入。

---

## 二、核心运行时 — Agent Loop

Claude Code 的心脏：用户输入 → LLM 调用 → 工具执行 → 响应输出的核心循环。

| 文档 | 说明 | 规模 |
|------|------|------|
| [[LLM_INTERACTION\|LLM 交互]] | API 客户端、模型选择、请求构建、流式响应、重试机制、成本追踪、prompt caching | ~660 行 |
| [[CONVERSATION_MANAGEMENT\|对话管理]] | QueryEngine 状态机、消息历史、自动压缩 (circuit breaker)、会话持久化与恢复 | ~1,290 行 |
| [[PROMPT_MANAGEMENT\|Prompt 管理]] | System Prompt 构建、多源 Prompt 合并、动态注入、effort 级别、thinking 配置 | ~940 行 |
| [[MEMORY_MANAGEMENT\|记忆管理]] | Memory Directory (index+retrieval)、AutoDream 离线整理、上下文溢出、工具结果 spilling | ~1,060 行 |

**阅读建议**：[[LLM_INTERACTION\|LLM 交互]] 是最底层（API 调用），[[CONVERSATION_MANAGEMENT\|对话管理]] 是状态管理层，[[PROMPT_MANAGEMENT\|Prompt 管理]] 是输入构建层，[[MEMORY_MANAGEMENT\|记忆管理]] 是长期记忆层。按此顺序阅读。

---

## 三、工具与 Agent 系统

Claude 的"手"和"分身"——执行能力的核心。

| 文档 | 说明 | 规模 |
|------|------|------|
| [[TOOL_SYSTEM\|工具系统]] | Tool 接口定义、40+ 内置工具、工具注册/过滤/执行管线、StreamingToolExecutor 流式执行 | ~860 行 |
| [[AGENT_SYSTEM\|Agent 系统]] | AgentTool 派生、AgentDefinition 多源加载、Task 生命周期、工具隔离、Coordinator/Swarm 模式 | ~740 行 |
| [[TASK_SYSTEM\|Task 系统]] | 两套独立任务系统：Runtime Task Framework (7 种后台任务) + Swarm Task List (文件锁协调)、磁盘输出、前后台切换 | ~480 行 |
| [[COMMAND_SYSTEM\|命令系统]] | 三种命令类型 (prompt/local/local-jsx)、懒加载、多源合并、onDone 编排 | ~770 行 |

**阅读建议**：[[TOOL_SYSTEM\|工具系统]] 是基础（所有能力的载体），[[AGENT_SYSTEM\|Agent 系统]] 建立在其之上（多 Agent 协作），[[TASK_SYSTEM\|Task 系统]] 是后台执行管理层，[[COMMAND_SYSTEM\|命令系统]] 是用户触发的斜杠命令入口。

---

## 四、扩展生态

第三方和自定义扩展能力——Hooks、Plugin、Skill、MCP 构成的扩展矩阵。

| 文档 | 说明 | 规模 |
|------|------|------|
| [[MCP_SYSTEM\|MCP 协议]] | Model Context Protocol 客户端、服务器发现/连接/重连、工具/资源/Prompt 集成、OAuth 认证 | ~1,280 行 |
| [[HOOKS_SYSTEM\|Hooks 系统]] | 6 种执行模式 (Command/HTTP/Prompt/Agent/SDK/Function)、27 种事件、权限集成、异步 Hook | ~890 行 |
| [[PLUGIN_SYSTEM\|Plugin 系统]] | Marketplace 生态、Manifest schema、加载生命周期、企业策略管控、依赖解析、热重载 | ~860 行 |
| [[SKILL_SYSTEM\|Skill 系统]] | Markdown 技能定义、SKILL.md 规范、Bundled/Plugin/User Skills、MCP-backed Skills | ~1,120 行 |

**阅读建议**：[[MCP_SYSTEM\|MCP 协议]] 是最基础的扩展协议，[[HOOKS_SYSTEM\|Hooks 系统]] 是生命周期切面，[[PLUGIN_SYSTEM\|Plugin 系统]] 是分发生态，[[SKILL_SYSTEM\|Skill 系统]] 是最终用户的技能包。按此顺序阅读。

---

## 五、运行模式

不同场景下的运行形态——从交互式 CLI 到 SDK 集成到远程执行。

| 文档 | 说明 | 规模 |
|------|------|------|
| [[SDK_SYSTEM\|SDK / Headless]] | NDJSON 协议、StructuredIO、control_request/response RPC、SDKMessage 25+ 类型、权限委托 | ~940 行 |
| [[SERVER_REMOTE_BRIDGE\|Server / Remote / Bridge]] | Direct Connect 服务器、Remote Control 桥接、CCR 远程会话、三种传输层 (SSE/Hybrid/WS) | ~730 行 |

**阅读建议**：[[SDK_SYSTEM\|SDK / Headless]] 是无头运行的基础协议，[[SERVER_REMOTE_BRIDGE\|Server / Remote / Bridge]] 是三种远程模式的完整文档。

---

## 六、UI 与可观测性

用户看到的和运维看到的。

| 文档 | 说明 | 规模 |
|------|------|------|
| [[RENDERING_SYSTEM\|渲染系统]] | React/Ink 终端 UI、组件树、权限对话框、Markdown 渲染、流式输出、主题系统 | ~1,030 行 |
| [[MONITORING_SYSTEM\|监控系统]] | Sentry 错误追踪、Datadog APM、GrowthBook Feature Flags、1P 事件日志、成本追踪、性能 Profiling | ~1,160 行 |

---

## 七、特色功能

独立的创新子系统。

| 文档 | 说明 | 规模 |
|------|------|------|
| [[KAIROS_SYSTEM\|KAIROS 主动助手]] | "Always-on" 监控模式、自动操作、GitHub Webhook、日报生成、通知系统 | ~1,070 行 |
| [[COMPUTER_USE\|Computer Use]] | macOS 屏幕交互 (代号 Chicago)：Swift 截图、Rust 键鼠、MCP 集成、会话锁、Escape 热键中止 | ~800 行 |

---

## 文档全景图

```
                    ┌─────────────────────────┐
                    │   ARCHITECTURE_OVERVIEW  │  ← 从这里开始
                    │   DESIGN_HIGHLIGHTS      │
                    │   SECURITY_ARCHITECTURE  │
                    └────────────┬────────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                  ▼
     ┌────────────────┐ ┌───────────────┐ ┌────────────────┐
     │ 核心运行时      │ │ 工具与 Agent  │ │ 扩展生态        │
     │                │ │               │ │                │
     │ LLM_INTERACTION│ │ TOOL_SYSTEM   │ │ MCP_SYSTEM     │
     │ CONVERSATION   │ │ AGENT_SYSTEM  │ │ HOOKS_SYSTEM   │
     │ PROMPT         │ │ TASK_SYSTEM   │ │ PLUGIN_SYSTEM  │
     │ MEMORY         │ │ COMMAND_SYSTEM│ │ SKILL_SYSTEM   │
     └───────┬────────┘ └───────┬───────┘ └───────┬────────┘
             │                  │                  │
             └──────────────────┼──────────────────┘
                                │
              ┌─────────────────┼─────────────────┐
              ▼                 ▼                  ▼
     ┌────────────────┐ ┌───────────────┐ ┌────────────────┐
     │ 运行模式        │ │ UI & 可观测性  │ │ 特色功能        │
     │                │ │               │ │                │
     │ SDK_SYSTEM     │ │ RENDERING     │ │ KAIROS_SYSTEM  │
     │ SERVER_REMOTE  │ │ MONITORING    │ │ COMPUTER_USE   │
     │ _BRIDGE        │ │               │ │                │
     └────────────────┘ └───────────────┘ └────────────────┘
```

---

## 推荐阅读路径

### 路径 A：快速了解（~3 篇）

1. [[ARCHITECTURE_OVERVIEW\|架构概览]] — 整体架构
2. [[DESIGN_HIGHLIGHTS\|设计亮点]] — 工程亮点
3. [[SECURITY_ARCHITECTURE\|安全架构]] — 安全模型

### 路径 B：理解核心循环（~8 篇）

1. [[ARCHITECTURE_OVERVIEW\|架构概览]] → 2. [[LLM_INTERACTION\|LLM 交互]] → 3. [[CONVERSATION_MANAGEMENT\|对话管理]] → 4. [[TOOL_SYSTEM\|工具系统]] → 5. [[AGENT_SYSTEM\|Agent 系统]] → 6. [[TASK_SYSTEM\|Task 系统]] → 7. [[PROMPT_MANAGEMENT\|Prompt 管理]] → 8. [[MEMORY_MANAGEMENT\|记忆管理]]

### 路径 C：扩展开发者（~4 篇）

1. [[MCP_SYSTEM\|MCP 协议]] → 2. [[HOOKS_SYSTEM\|Hooks 系统]] → 3. [[PLUGIN_SYSTEM\|Plugin 系统]] → 4. [[SKILL_SYSTEM\|Skill 系统]]

### 路径 D：集成/部署（~3 篇）

1. [[SDK_SYSTEM\|SDK / Headless]] → 2. [[SERVER_REMOTE_BRIDGE\|Server / Remote / Bridge]] → 3. [[MONITORING_SYSTEM\|监控系统]]

### 路径 E：全量阅读（建议顺序）

按本文档的章节顺序 I → VII 依次阅读，即：全局架构 → 核心运行时 → 工具 Agent → 扩展生态 → 运行模式 → UI 监控 → 特色功能。

---

## 统计

| 指标 | 数值 |
|------|------|
| 文档总数 | 21 篇 |
| 总行数 | ~18,500 行 |
| 覆盖模块 | 35+ 个 `src/` 子目录 |
| 平均篇幅 | ~880 行/篇 |
| 最大篇幅 | [[CONVERSATION_MANAGEMENT\|对话管理]] (~1,290 行) |
| 最小篇幅 | [[TASK_SYSTEM\|Task 系统]] (~480 行) |
