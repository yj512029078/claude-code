---
title: Claude Code 记忆管理系统
aliases: [记忆管理, Memory Management]
series: Claude Code 架构解析
category: 核心运行时
order: 7
tags:
  - claude-code
  - memory
  - context-window
  - auto-dream
  - spilling
date: 2026-04-04
---

# Claude Code 记忆管理系统 — 技术文档

## 目录

1. [架构概览](#1-架构概览)
2. [七层记忆文件全景](#2-七层记忆文件全景)
   - 2.1 [记忆文件清单](#21-记忆文件清单)
   - 2.2 [各层更新时机详解](#22-各层更新时机详解)
   - 2.3 [一次对话中的记忆更新时间线](#23-一次对话中的记忆更新时间线)
3. [短期记忆：对话上下文管理](#3-短期记忆对话上下文管理)
   - 3.1 [消息数组与 Compact Boundary](#31-消息数组与-compact-boundary)
   - 3.2 [Token 计数机制](#32-token-计数机制)
   - 3.3 [自动压缩 (Auto Compact)](#33-自动压缩-auto-compact)
   - 3.4 [完整压缩 (Full Compaction)](#34-完整压缩-full-compaction)
   - 3.5 [Session Memory Compact](#35-session-memory-compact)
   - 3.6 [部分压缩 (Partial Compact)](#36-部分压缩-partial-compact)
   - 3.7 [微压缩 (Microcompact)](#37-微压缩-microcompact)
   - 3.8 [History Snip](#38-history-snip)
   - 3.9 [Context Collapse](#39-context-collapse)
   - 3.10 [响应式压缩 (Reactive Compact)](#310-响应式压缩-reactive-compact)
4. [中期记忆：Session Memory](#4-中期记忆session-memory)
   - 4.1 [设计思想](#41-设计思想)
   - 4.2 [提取触发条件](#42-提取触发条件)
   - 4.3 [提取流程](#43-提取流程)
   - 4.4 [配置参数](#44-配置参数)
5. [长期记忆：项目上下文文件](#5-长期记忆项目上下文文件)
   - 5.1 [CLAUDE.md 文件体系](#51-claudemd-文件体系)
   - 5.2 [Auto Memory (memdir)](#52-auto-memory-memdir)
   - 5.3 [Team Memory](#53-team-memory)
   - 5.4 [Agent Memory](#54-agent-memory)
6. [会话持久化与恢复](#6-会话持久化与恢复)
   - 6.1 [JSONL 转录文件](#61-jsonl-转录文件)
   - 6.2 [Session Resume 机制](#62-session-resume-机制)
   - 6.3 [Tool Result 溢出存储](#63-tool-result-溢出存储)
7. [查询管线中的记忆处理流水线](#7-查询管线中的记忆处理流水线)
8. [压缩后的清理与缓存失效](#8-压缩后的清理与缓存失效)
9. [子代理记忆管理](#9-子代理记忆管理)
10. [配置参考](#10-配置参考)
11. [核心数据结构](#11-核心数据结构)
12. [关键源文件索引](#12-关键源文件索引)

---

## 1. 架构概览

Claude Code 的记忆管理系统是一个**多层级**、**多策略**的上下文管理架构，旨在解决 LLM 有限上下文窗口与无限对话长度之间的矛盾。整体设计遵循以下核心原则：

- **分层管理**：短期（当前对话轮次）、中期（Session Memory 笔记文件）、长期（CLAUDE.md 项目记忆）各层独立运作
- **渐进压缩**：从轻量级的微压缩到重量级的全量摘要，逐级应对上下文增长
- **无损语义**：压缩以 LLM 摘要为核心手段，而非简单截断，尽量保留对话语义
- **透明持久化**：对话以 JSONL 格式持久存储，支持跨会话恢复

```
┌─────────────────────────────────────────────────────────────────┐
│                        查询管线 (query.ts)                       │
│                                                                   │
│  Message[] ──► Compact Boundary Slice                            │
│            ──► Tool Result Budget                                 │
│            ──► History Snip (feature flag)                        │
│            ──► Microcompact                                       │
│            ──► Context Collapse (feature flag)                    │
│            ──► Auto Compact (threshold check)                    │
│            ──► normalize → API call                               │
│                                                                   │
├───────────────────────┬───────────────────────────────────────────┤
│  Session Memory       │  CLAUDE.md / Auto Memory / Agent Memory   │
│  (background notes)   │  (persistent project context)             │
├───────────────────────┴───────────────────────────────────────────┤
│  JSONL Transcript Persistence (session storage)                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. 七层记忆文件全景

系统共有 **7 层磁盘记忆文件**（实际写入磁盘的文件），分为 3 大类：手动维护的指令文件、AI 自动提取的记忆文件、会话级笔记文件。

### 2.1 记忆文件清单

| # | 层级 | 文件路径 | 谁写入 | 跨会话 | 注入给模型 |
|---|------|----------|--------|--------|------------|
| **1** | Managed Memory | `/etc/claude-code/CLAUDE.md` | 系统管理员手动 | 是 | 是 (userContext) |
| **2** | User Memory | `~/.claude/CLAUDE.md` | 用户手动 | 是 | 是 (userContext) |
| **3** | Project Memory | `CLAUDE.md`, `.claude/CLAUDE.md`, `.claude/rules/*.md` | 用户/团队手动 | 是 (提交 git) | 是 (userContext) |
| **4** | Local Memory | `CLAUDE.local.md` (项目根目录) | 用户手动 | 是 (不提交 git) | 是 (userContext) |
| **5** | Auto Memory | `~/.claude/projects/<cwd>/memory/MEMORY.md` + 子文件 | AI 后台自动 + 主对话 AI 主动 | 是 | 是 (system prompt) |
| **6** | Team Memory | `<project>/.claude/team-memory/*.md` | AI 后台自动 | 是 (可提交 git) | 是 (system prompt) |
| **7** | Session Memory | session 目录下的 `summary.md` | AI 后台自动 | 否 (会话级) | 间接 (压缩时用作摘要) |

另有一个非记忆但相关的持久层：
- **JSONL 转录文件** (`~/.claude/projects/<dir>/<session-id>.jsonl`) — 完整对话记录，用于 `--resume` 恢复，不是注入给模型的"记忆"

### 2.2 各层更新时机详解

#### 第 1–4 层：CLAUDE.md 体系 — 手动更新，每次 API 调用前读取

这 4 层是**纯被动**的，Claude 在对话时**只读取不写入**（除非用户明确要求编辑）：

```
加载时机: 每次 API 调用前 (getUserContext() → getMemoryFiles() → 封装为 <system-reminder> 注入)
更新时机: 用户手动编辑（或 AI 被明确要求时通过 FileEdit/FileWrite 写入）
缓存机制: getMemoryFiles() 使用 memoize 缓存
缓存失效: compact 后调用 resetGetMemoryFilesCache('compact') + getUserContext.cache.clear()
```

#### 第 5 层：Auto Memory — 每轮对话结束后后台提取

**文件**: `services/extractMemories/extractMemories.ts`

通过 `handleStopHooks` 在**每个完整查询循环结束时**（模型最终回复无 tool_use 时）触发：

```
initExtractMemories() 注册为 stopHook
                ↓
每次模型给出最终回复(无tool_use)时触发
                ↓
前置检查:
  ├─ isAutoMemoryEnabled() (默认开启)
  ├─ isExtractModeActive() (feature flag tengu_passport_quail)
  └─ querySource === 'repl_main_thread'
                ↓
频率控制: 每 N 个 eligible turn 运行一次 (tengu_bramble_lintel, 默认 1)
                ↓
互斥检查: 主对话中 AI 已经写过 memory 文件? → 跳过
                ↓
运行 forked agent (最多 5 轮):
  ├─ 扫描已有记忆文件 (scanMemoryFiles → formatMemoryManifest)
  ├─ 读取对话中的新消息 (cursor: lastMemoryMessageUuid)
  ├─ 允许的工具: Read, Grep, Glob, readonly Bash, Edit/Write (仅 memory 目录内)
  └─ 通过 FileEdit/FileWrite 写入/更新/删除记忆条目
```

**记忆条目格式**: 每条记忆是独立 `.md` 文件，带 frontmatter:
```markdown
---
name: {{memory name}}
description: {{one-line description}}
type: {{user, feedback, project, reference}}
---

{{memory content}}
```

**四种记忆类型**:
- `user` — 用户角色、偏好、知识水平（如 "高级 Go 开发者，React 新手"）
- `feedback` — 用户对工作方式的纠正和确认（如 "测试中不要 mock 数据库"）
- `project` — 项目动态、目标、截止日期（如 "周四后冻结非关键合并"）
- `reference` — 外部系统指向（如 "pipeline bugs 追踪在 Linear 的 INGEST 项目"）

**不该保存的**: 代码模式、架构、git 历史、调试方案 — 这些可从当前代码推导。

**入口文件**: `MEMORY.md` 是索引文件，限制 200 行 / 25KB (`MAX_ENTRYPOINT_LINES`, `MAX_ENTRYPOINT_BYTES`)。

**主对话互斥**: 主对话中的 AI 也有能力直接写 memory 文件（system prompt 中包含保存指令）。当 `extractMemories` 检测到主对话已写过 memory 文件 (`hasMemoryWritesSince`)，会跳过后台提取并推进 cursor，避免重复。

#### 第 6 层：Team Memory — 随 Auto Memory 同步

**文件**: `memdir/teamMemPaths.ts`, `services/teamMemorySync/`

当 `TEAMMEM` feature flag 启用时，`extractMemories` 的 forked agent 同时写入两个目录：

| 方向 | 路径 | 可见性 |
|------|------|--------|
| Private (个人) | `~/.claude/projects/<cwd>/memory/` | 仅本人 |
| Team (团队) | `<project>/.claude/team-memory/` | 可提交 git，团队共享 |

**更新时机**: 与 Auto Memory 完全同步 — 在 `extractMemories` 的同一次 forked agent 运行中决定写入 private 还是 team（或两者）。

**安全防护**: Team memory 有专门的 secret scanner (`teamMemSecretGuard.ts`)，防止敏感信息（API key 等）被写入可能提交到 git 的团队记忆文件。

**Scope 决策规则**（在 prompt 中指导 AI）：
- `user` 类型 → 始终 private
- `feedback` 类型 → 默认 private，除非明显是项目级约定
- `project` 类型 → 偏向 team
- `reference` 类型 → 通常 team

#### 第 7 层：Session Memory — 对话中周期性后台提取

**文件**: `services/SessionMemory/sessionMemory.ts`

与 Auto Memory 不同，Session Memory 在**对话进行中**持续更新，是一个**会话级**笔记文件。

```
注册为 postSamplingHook (每次模型采样后检查)
                ↓
首次: tokenCountWithEstimation >= 10,000 → 初始化
                ↓
后续更新需同时满足:
  ① token 增长 >= 5,000 (硬性要求, 防止过度频繁)
  ② 工具调用 >= 3 次  OR  最后一轮无工具调用(自然断点)
                ↓
运行 forked agent → 只允许 FileEdit 更新 summary.md
```

**与 Auto Memory 的关键区别**:

| 维度 | Session Memory (第7层) | Auto Memory (第5层) |
|------|------------------------|---------------------|
| **生命周期** | 当前会话 | 跨会话永久 |
| **触发时机** | 对话中每 ~5K tokens 增长 | 每轮对话结束 |
| **内容性质** | 会话摘要/进度笔记 | 结构化记忆条目 (带 frontmatter) |
| **主要用途** | 压缩时替代 LLM 摘要 (SM Compact) | 下次对话时提供历史上下文 |
| **文件数** | 1 个 summary.md | 多个主题 .md 文件 + MEMORY.md 索引 |
| **注入方式** | 间接 — compact 时作为 summary 注入 | 直接 — 作为 system prompt 的一部分 |
| **钩子类型** | postSamplingHook | stopHook |

### 2.3 一次对话中的记忆更新时间线

```
会话开始
  │
  ├─ [读取] CLAUDE.md 体系 (第1-4层) → 注入 userContext
  ├─ [读取] MEMORY.md + 记忆条目 (第5层) → 注入 system prompt
  ├─ [读取] Team Memory (第6层, 如启用) → 注入 system prompt
  │
  ├─ Turn 1-3: 对话进行，token 累积
  │    ├─ 每轮结束: extractMemories 检查 → token 不足，跳过
  │    └─ [无记忆文件更新]
  │
  ├─ Turn 4: token 达到 ~10,000
  │    ├─ Session Memory 初始化，首次提取 → summary.md 创建 (第7层)
  │    └─ 该轮结束: extractMemories 运行 → 可能写入 memory/*.md (第5/6层)
  │
  ├─ Turn 5-8: 继续对话
  │    ├─ Session Memory: token 增长 5K + 3次 tool call → 更新 summary.md (第7层)
  │    ├─ extractMemories: 每轮结束检查一次 → 有新见解则写入 (第5/6层)
  │    └─ CLAUDE.md: 仅读取，不更新 (第1-4层)
  │
  ├─ Turn 10: token 接近阈值 (~167K)
  │    ├─ 自动压缩触发:
  │    │    ├─ 优先: 用 Session Memory (summary.md) 做免 API 压缩 (SM Compact)
  │    │    └─ 回退: 用 LLM 生成摘要 (Full Compaction)
  │    └─ 压缩后: 清除 CLAUDE.md 缓存 → 下次 API 调用重新加载第1-4层
  │
  ├─ Turn 11+: 压缩后继续
  │    ├─ Session Memory: 重置 lastSummarizedMessageId，继续周期性提取
  │    ├─ extractMemories: 继续每轮结束检查
  │    └─ 可能再次触发压缩 (循环)
  │
  └─ 会话结束
       ├─ 所有记忆文件保留在磁盘
       ├─ 第5/6层: 下次新会话立即可用
       ├─ 第7层: 仅通过 --resume 恢复时可用
       └─ JSONL 转录: 保留 30 天 (cleanupPeriodDays)
```

---

## 3. 短期记忆：对话上下文管理

### 3.1 消息数组与 Compact Boundary

**核心概念**：系统维护一个完整的 `Message[]` 数组作为对话历史。但发送给 API 时，只取 **最后一个 compact boundary 之后** 的消息。

**Compact Boundary** 是一个特殊的 `SystemCompactBoundaryMessage`，标记压缩发生的位置：

```typescript
// utils/messages.ts
function getMessagesAfterCompactBoundary<T extends Message>(
  messages: T[],
  options?: { includeSnipped?: boolean }
): T[] {
  const boundaryIndex = findLastCompactBoundaryIndex(messages)
  const sliced = boundaryIndex === -1 ? messages : messages.slice(boundaryIndex)
  // 可选: 应用 snip 投影（HISTORY_SNIP feature flag）
  if (!options?.includeSnipped && feature('HISTORY_SNIP')) {
    return projectSnippedView(sliced)
  }
  return sliced
}
```

**压缩后的消息重建顺序**（`buildPostCompactMessages`）：

```
[Compact Boundary] → [Summary Messages] → [Preserved Tail Messages] → [Attachments] → [Hook Results]
```

这不是固定大小的滑动窗口，而是**摘要 + 可选保留尾部** 的弹性方案。

### 3.2 Token 计数机制

Token 计数是触发所有压缩策略的核心度量。主要函数为 `tokenCountWithEstimation`（`utils/tokens.ts`）：

**算法**：
1. 从消息数组末尾向前搜索最后一条有 `usage` 数据的 assistant 消息
2. 处理共享 `message.id` 的分裂消息（并行 tool call 时同一 API 响应生成多条记录）
3. 取该消息的 `usage` 作为准确锚点：`input_tokens + cache_creation_input_tokens + cache_read_input_tokens + output_tokens`
4. 对锚点之后的消息使用粗略估算（`roughTokenCountEstimationForMessages`，约 4 字符/token）
5. 两者相加得到当前上下文大小估计

```typescript
// 粗略估算函数 (services/tokenEstimation.ts)
function roughTokenCountEstimation(text: string): number {
  return Math.ceil(text.length / 4) // ~4 bytes per token
}
```

### 3.3 自动压缩 (Auto Compact)

**文件**: `services/compact/autoCompact.ts`

自动压缩是最主要的上下文管理机制，当对话接近上下文窗口限制时自动触发。

#### 阈值计算

```typescript
const AUTOCOMPACT_BUFFER_TOKENS = 13_000

function getEffectiveContextWindowSize(model: string): number {
  const reservedForSummary = Math.min(
    getMaxOutputTokensForModel(model),
    20_000  // MAX_OUTPUT_TOKENS_FOR_SUMMARY
  )
  let contextWindow = getContextWindowForModel(model, betas)
  // 可通过环境变量限制
  if (process.env.CLAUDE_CODE_AUTO_COMPACT_WINDOW) {
    contextWindow = Math.min(contextWindow, parsed)
  }
  return contextWindow - reservedForSummary
}

function getAutoCompactThreshold(model: string): number {
  const effective = getEffectiveContextWindowSize(model)
  const threshold = effective - AUTOCOMPACT_BUFFER_TOKENS
  // 支持百分比覆盖（用于测试）
  if (process.env.CLAUDE_AUTOCOMPACT_PCT_OVERRIDE) {
    return Math.min(percentageThreshold, threshold)
  }
  return threshold
}
```

**示例计算**（200K 上下文窗口模型）：
- 有效窗口 = 200,000 - 20,000 = 180,000
- 自动压缩阈值 = 180,000 - 13,000 = **167,000 tokens**

#### Token 警告状态

系统定义了多个警告级别：

| 状态 | 计算方式 | 作用 |
|------|----------|------|
| Warning | `threshold - 20,000` | UI 提示即将达到限制 |
| Error | `threshold - 20,000` | UI 显示错误级警告 |
| Auto Compact | `threshold` | 触发自动压缩 |
| Blocking | `effective - 3,000` | 阻止进一步输入，留出手动 `/compact` 空间 |

#### 执行流程

```
autoCompactIfNeeded()
    ├── 检查熔断器 (连续失败 >= 3 次则跳过)
    ├── shouldAutoCompact() 判断
    │   ├── 排除递归调用 (session_memory, compact, marble_origami)
    │   ├── 检查 isAutoCompactEnabled()
    │   ├── 排除 REACTIVE_COMPACT / CONTEXT_COLLAPSE 模式
    │   └── tokenCountWithEstimation() >= threshold ?
    │
    ├── 优先尝试 Session Memory Compact
    │   ├── 成功 → 重置 lastSummarizedMessageId, 清理, 返回
    │   └── 失败 → fallback
    │
    └── 执行 compactConversation() (完整压缩)
        ├── 成功 → 重置, 清理, 返回
        └── 失败 → 累加 consecutiveFailures
```

**熔断器机制**：连续失败 3 次后停止尝试自动压缩，防止在上下文不可恢复时浪费 API 调用。据统计，曾有 1,279 个会话经历 50+ 次连续失败，每天全球浪费约 250K 次 API 调用。

### 3.4 完整压缩 (Full Compaction)

**文件**: `services/compact/compact.ts`

完整压缩通过 LLM 生成对话摘要来实现上下文缩减。

#### 流程

1. **Pre-Compact Hooks**: 执行注册的预压缩钩子，获取自定义指令
2. **消息预处理**:
   - `stripImagesFromMessages()`: 移除图片（替换为 `[image]` 标记）
   - `stripReinjectedAttachments()`: 移除会被重新注入的附件（skill_discovery 等）
   - `getMessagesAfterCompactBoundary()`: 只取边界后的消息
3. **LLM 摘要调用**:
   - 优先使用 `runForkedAgent()` 复用主对话的 prompt cache
   - 回退到 `queryModelWithStreaming()` 直接流式调用
   - 系统提示: `"You are a helpful AI assistant tasked with summarizing conversations."`
   - 关闭 thinking, 限制 max_output_tokens
4. **Prompt-Too-Long 重试**: 如果摘要请求本身超长，截断最老的 API round 组重试（最多 3 次）
5. **构建压缩结果**:
   - 创建 `CompactBoundaryMessage`
   - 保存已发现的延迟加载工具名（`preCompactDiscoveredTools`）
   - 生成摘要 User Message（标记为 `isCompactSummary: true`）
6. **恢复附件**:
   - 最近访问的文件（最多 5 个，总计 50K token 预算，单个文件 5K token 上限）
   - Plan 文件、Plan Mode 指令
   - 已调用的 Skill 内容（单个 5K token，总计 25K token 预算）
   - 延迟工具 delta、Agent listing delta、MCP 指令 delta
   - 异步 Agent 状态
7. **Post-Compact Hooks**: 执行 SessionStart hooks 和 PostCompact hooks
8. **缓存重置与遥测**: 通知 prompt cache 检测器、重追加 session metadata

#### Token 预算常量

```typescript
POST_COMPACT_MAX_FILES_TO_RESTORE = 5
POST_COMPACT_TOKEN_BUDGET = 50_000
POST_COMPACT_MAX_TOKENS_PER_FILE = 5_000
POST_COMPACT_MAX_TOKENS_PER_SKILL = 5_000
POST_COMPACT_SKILLS_TOKEN_BUDGET = 25_000
```

### 3.5 Session Memory Compact

**文件**: `services/compact/sessionMemoryCompact.ts`

这是一种**免 API 调用**的压缩策略，直接用 Session Memory 文件内容作为摘要，跳过 LLM 摘要步骤。

#### 原理

如果 Session Memory 后台提取器已经将对话要点记录到 `summary.md`，则可以：
1. 读取 Session Memory 文件作为摘要
2. 保留 `lastSummarizedMessageId` 之后的消息作为尾部
3. 构建 compact boundary + summary + preserved tail

#### 保留消息计算

```typescript
const DEFAULT_SM_COMPACT_CONFIG = {
  minTokens: 10_000,      // 最少保留 10K tokens
  minTextBlockMessages: 5, // 至少保留 5 条有文本块的消息
  maxTokens: 40_000,       // 最多保留 40K tokens
}
```

`calculateMessagesToKeepIndex()` 从 `lastSummarizedMessageId` 开始，向前扩展直到满足最小阈值或达到最大上限。同时确保不会拆分 `tool_use`/`tool_result` 对和共享 `message.id` 的 thinking block。

#### 前置条件

- `tengu_session_memory` 和 `tengu_sm_compact` feature flags 均开启
- Session Memory 文件存在且有实际内容（非空模板）
- 压缩后的总 token 不超过自动压缩阈值

#### Session Memory 截断

为防止 Session Memory 内容占满压缩后的 token 预算，按 section 截断：

```typescript
MAX_SECTION_LENGTH = 2000     // 每个 section 最大 token
MAX_TOTAL_SESSION_MEMORY_TOKENS = 12000  // 总上限
```

### 3.6 部分压缩 (Partial Compact)

**文件**: `services/compact/compact.ts` → `partialCompactConversation()`

支持用户通过 UI 选择一个消息锚点，只压缩锚点前或后的部分：

- **`from` 方向**: 压缩锚点之后的消息，保留之前的（保护 prompt cache 前缀）
- **`up_to` 方向**: 压缩锚点之前的消息，保留之后的（会导致 prompt cache 失效）

### 3.7 微压缩 (Microcompact)

**文件**: `services/compact/microCompact.ts`

微压缩是一种轻量级的工具结果清理机制，在完整压缩之前运行。

#### 可压缩的工具

```typescript
const COMPACTABLE_TOOLS = new Set([
  'Read',          // 文件读取
  'Bash', 'Shell', // Shell 工具
  'Grep',          // 搜索
  'Glob',          // 文件匹配
  'WebSearch',     // Web 搜索
  'WebFetch',      // Web 抓取
  'Edit',          // 文件编辑
  'Write',         // 文件写入
])
```

#### 三种路径

1. **Time-Based Microcompact**: 当距离上次 assistant 消息超过阈值（cache 已过期），直接清空旧 tool_result 内容为 `[Old tool result content cleared]`。保留最近 N 个。

2. **Cached Microcompact** (feature-gated): 使用 API 的 `cache_edits` 能力在服务端删除旧 tool result，不修改本地消息内容，不破坏 prompt cache。

3. **Legacy Path**: 已移除（`tengu_cache_plum_violet` 始终为 true）。

#### Token 估算

`estimateMessageTokens()` 遍历消息中的所有 content block，使用 `roughTokenCountEstimation` 估算并乘以 4/3 的保守系数。

### 3.8 History Snip

**Feature Flag**: `HISTORY_SNIP`

Snip 是一种投影机制，为模型提供经过裁剪的历史视图，同时 REPL 保留完整滚动历史。通过 `snipProjection.ts` 的 `projectSnippedView()` 函数实现。

Snip 释放的 token 会传递给自动压缩计算：`snipTokensFreed` 参数让 `shouldAutoCompact` 能正确评估经过 snip 后的实际上下文大小。

### 3.9 Context Collapse

**Feature Flag**: `CONTEXT_COLLAPSE`

Context Collapse 是一种替代性的上下文管理系统。当启用时，它**接管**自动压缩的角色：

- 90% 有效窗口: 触发 context commit
- 95% 有效窗口: 触发 blocking spawn

自动压缩的阈值（约 93%）正好落在这两个阈值之间，为防止竞态条件，Context Collapse 开启时自动压缩被抑制。

### 3.10 响应式压缩 (Reactive Compact)

**Feature Flag**: `REACTIVE_COMPACT`

响应式压缩在 API 返回 `prompt_too_long` 错误时被动触发，作为最后的保护机制：

1. 流式响应中检测到被扣留的 `prompt_too_long` 错误
2. 尝试 Context Collapse drain（如果启用）
3. 回退到响应式压缩
4. 重试 API 调用

---

## 4. 中期记忆：Session Memory

### 4.1 设计思想

**文件**: `services/SessionMemory/sessionMemory.ts`

Session Memory 是一个**后台运行**的记忆提取系统。它通过注册 `postSamplingHook`，在每次模型采样后检查是否需要提取，然后使用 forked subagent 将对话要点写入一个 Markdown 文件（`summary.md`），全程不中断主对话。

```
主对话循环 ──► postSamplingHook ──► shouldExtractMemory()? 
                                          │
                                    Yes ──► 创建隔离的 subagent 上下文
                                          │
                                          ├── 读取当前 summary.md
                                          ├── 构建更新 prompt
                                          └── runForkedAgent() 用 FileEdit 更新文件
```

### 4.2 提取触发条件

```typescript
function shouldExtractMemory(messages: Message[]): boolean {
  const currentTokenCount = tokenCountWithEstimation(messages)
  
  // 1. 首次初始化：需要达到 minimumMessageTokensToInit (默认 10,000)
  if (!isSessionMemoryInitialized()) {
    if (currentTokenCount < minimumMessageTokensToInit) return false
    markSessionMemoryInitialized()
  }

  // 2. 增量更新条件 (两者必须同时满足 OR 对话自然断点):
  const hasMetTokenThreshold = 
    (currentTokenCount - tokensAtLastExtraction) >= minimumTokensBetweenUpdate
  const hasMetToolCallThreshold = 
    countToolCallsSince(messages, lastMemoryMessageUuid) >= toolCallsBetweenUpdates
  
  // Token 阈值是硬性要求，防止过度频繁提取
  return (hasMetTokenThreshold && hasMetToolCallThreshold) ||
         (hasMetTokenThreshold && !hasToolCallsInLastAssistantTurn(messages))
}
```

**"自然断点"条件**: 如果最后一个 assistant 轮次没有工具调用（纯文本回复），且达到了 token 阈值，也会触发提取。这捕获了对话中的自然暂停点。

### 4.3 提取流程

1. **初始化文件**: 在 session memory 目录创建 `summary.md`（权限 0o600），首次创建时写入模板
2. **隔离上下文**: 创建 `subagentContext` 避免污染主对话的文件读取缓存
3. **读取现有内容**: 通过 `FileReadTool` 读取当前 summary.md
4. **构建 prompt**: `buildSessionMemoryUpdatePrompt()` 生成更新指令
5. **运行 forked agent**: 使用 `runForkedAgent()` 执行提取
   - 只允许 `FileEditTool` 对 memory 文件的写入操作
   - 复用主对话的 prompt cache
6. **记录状态**: 更新 `tokensAtLastExtraction`、`lastSummarizedMessageId`

### 4.4 配置参数

```typescript
const DEFAULT_SESSION_MEMORY_CONFIG = {
  minimumMessageTokensToInit: 10_000,    // 初始化阈值
  minimumTokensBetweenUpdate: 5_000,     // 更新间隔 (token 增长)
  toolCallsBetweenUpdates: 3,            // 更新间隔 (工具调用数)
}
```

可通过 GrowthBook feature flag `tengu_sm_config` 远程覆盖。初始化时使用缓存值（非阻塞），后续延迟加载。

---

## 5. 长期记忆：项目上下文文件

### 5.1 CLAUDE.md 文件体系

**文件**: `utils/claudemd.ts`

CLAUDE.md 是 Claude Code 的项目级指令系统，按优先级从低到高加载：

| 优先级 | 类型 | 路径 | 说明 |
|--------|------|------|------|
| 1 (最低) | Managed | `/etc/claude-code/CLAUDE.md` | 全局管理员指令 |
| 2 | User | `~/.claude/CLAUDE.md` | 用户级全局指令 |
| 3 | Project | `CLAUDE.md`, `.claude/CLAUDE.md`, `.claude/rules/*.md` | 项目级指令（提交到版本控制） |
| 4 (最高) | Local | `CLAUDE.local.md` | 本地项目指令（不提交） |

**发现规则**:
- User memory 从用户主目录加载
- Project / Local 文件从当前目录向上遍历到根目录
- 距离当前目录越近的文件优先级越高（后加载）
- `.claude/rules/` 中的所有 `.md` 文件都会被扫描

**@include 指令**:
- 支持 `@path`, `@./relative`, `@~/home`, `@/absolute` 语法
- 只在叶文本节点中解析（不在代码块内）
- 被包含的文件作为独立条目在包含文件之前加入
- 防止循环引用（跟踪已处理文件）
- 不存在的文件静默忽略

**缓存策略**:
- `getMemoryFiles()` 使用 memoize 缓存
- `getUserContext()` 是外层 memoized 包装
- 压缩后通过 `resetGetMemoryFilesCache('compact')` 和 `getUserContext.cache.clear()` 重置

**注入方式**:
- 作为 `userContext` 的一部分，在 API 调用时 `prependUserContext()` 将其封装在 `<system-reminder>` XML 标签中，作为首条 user message 注入

### 5.2 Auto Memory (memdir)

**文件**: `memdir/paths.ts`, `memdir/memdir.ts`, `memdir/memoryTypes.ts`, `services/extractMemories/extractMemories.ts`

自动记忆是一个结构化的记忆目录，默认位于：

```
~/.claude/projects/<sanitized-cwd>/memory/
```

**路径解析优先级** (`getAutoMemPath`):
1. `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` 环境变量（Cowork 全路径覆盖）
2. `autoMemoryDirectory` 设置项（支持 `~/` 展开，仅 policy/local/user settings 生效，排除 projectSettings 防止恶意仓库劫持路径）
3. `<memoryBase>/projects/<sanitized-git-root>/memory/`（默认，memoryBase = `CLAUDE_CODE_REMOTE_MEMORY_DIR` 或 `~/.claude`）

同一 git 仓库的所有 worktree 共享同一个 auto-memory 目录（通过 `findCanonicalGitRoot` 统一）。

**入口文件**: `MEMORY.md` — 索引文件，限制 200 行 / 25KB。超出时按行截断并追加警告。

**记忆条目**: 每条记忆是独立的 `.md` 文件，带 frontmatter:
```markdown
---
name: {{memory name}}
description: {{one-line description — 用于未来对话判断相关性}}
type: {{user, feedback, project, reference}}
---

{{memory content — feedback/project 类型建议包含 **Why:** 和 **How to apply:** 行}}
```

**四种记忆类型及保存时机**:

| 类型 | 内容 | 何时保存 | 示例 |
|------|------|----------|------|
| `user` | 用户角色、偏好、知识水平 | 了解到用户身份/背景时 | "高级 Go 开发者，React 新手" |
| `feedback` | 用户纠正或确认的工作方式 | 用户纠正 ("不要这样") 或确认非显然做法时 | "测试中不要 mock 数据库，因为上季度出过事故" |
| `project` | 项目动态、目标、截止日期 | 了解到谁在做什么、为什么、到什么时候 | "周四后冻结非关键合并，移动端要切分支" |
| `reference` | 外部系统指向 | 了解到外部资源位置 | "pipeline bugs 追踪在 Linear 的 INGEST 项目" |

**什么不该保存** (在 prompt 中明确排除):
- 代码模式、架构、文件路径 — 可从代码推导
- Git 历史 — `git log` / `git blame` 是权威源
- 调试方案 — 修复在代码里，commit message 有上下文
- CLAUDE.md 中已有的内容
- 临时任务状态、当前对话上下文

**后台提取的更新机制** (`extractMemories`):
- 通过 `handleStopHooks` 在每轮对话最终回复后触发
- Forked agent 最多运行 5 轮（典型 2-4 轮: read → write）
- Agent 可以 Read/Grep/Glob 任意文件、只读 Bash，但只能 Edit/Write memory 目录内的文件
- 先注入已有记忆文件清单（`formatMemoryManifest`），避免浪费一轮 `ls`
- 与主对话互斥：`hasMemoryWritesSince` 检测到主对话已写 memory 文件则跳过

**回忆时的防过时机制**: 记忆可能过时，prompt 中要求 AI 在使用记忆前先验证（读文件或 grep 确认函数/文件仍存在），如有冲突以当前状态为准并更新/删除过时记忆。

**控制开关** (`isAutoMemoryEnabled`，优先级从高到低):
1. `CLAUDE_CODE_DISABLE_AUTO_MEMORY` 环境变量（1/true → 关）
2. `CLAUDE_CODE_SIMPLE` / `--bare` → 关
3. CCR 远程模式无 `CLAUDE_CODE_REMOTE_MEMORY_DIR` → 关
4. `autoMemoryEnabled` 设置项（支持项目级 opt-out）
5. 默认: 开启

**Kairos 模式 (assistant mode)**: 不维护 MEMORY.md 实时索引，改为追加日期日志文件 (`getAutoMemDailyLogPath` → `<autoMemPath>/logs/YYYY/MM/YYYY-MM-DD.md`)，由 nightly `/dream` skill 蒸馏为主题文件 + MEMORY.md。

### 5.3 Team Memory

**文件**: `memdir/teamMemPaths.ts`, `services/teamMemorySync/`, `services/extractMemories/extractMemories.ts`

Team Memory 是 Auto Memory 的团队共享扩展，当 `TEAMMEM` feature flag 启用时激活。

**存储路径**: `<project>/.claude/team-memory/`（可提交 git，团队成员共享）

**与 Auto Memory 的关系**: `extractMemories` 的 forked agent 在一次运行中同时决定写入：
- **Private**: `~/.claude/projects/<cwd>/memory/` — 个人记忆
- **Team**: `<project>/.claude/team-memory/` — 团队记忆

**Scope 决策规则** (在 extraction prompt 中指导):
- `user` 类型 → 始终 private（`<scope>always private</scope>`）
- `feedback` 类型 → 默认 private，除非明确是项目级约定（如测试策略）
- `project` 类型 → 偏向 team（`<scope>private or team, but strongly bias toward team</scope>`）
- `reference` 类型 → 通常 team

**安全防护**:
- Secret scanner (`teamMemSecretGuard.ts`, `secretScanner.ts`) 在写入前扫描内容，防止 API key、密码等被写入可能提交到 git 的文件
- 路径验证 (`teamMemPaths.ts`) 防止路径遍历攻击，检查 null byte、URL 编码遍历、Unicode 归一化攻击、反斜杠注入

**同步机制** (`services/teamMemorySync/`):
- Watcher 监听 team memory 目录变化
- 支持跨成员同步

### 5.4 Agent Memory

**文件**: `tools/AgentTool/agentMemory.ts`

每个 agent 类型有独立的持久化目录，按 scope 分三级：

| Scope | 路径 | 用途 |
|-------|------|------|
| `user` | `<memoryBase>/agent-memory/<agentType>/` | 用户全局，跨项目 |
| `project` | `<cwd>/.claude/agent-memory/<agentType>/` | 项目级，可提交 git |
| `local` | `<cwd>/.claude/agent-memory-local/<agentType>/` | 本地项目级，不提交 |

Agent type 名称中的冒号（如 `my-plugin:my-agent`）会被替换为短横线以兼容文件系统。

远程模式下 (`CLAUDE_CODE_REMOTE_MEMORY_DIR`)，local scope 重定向到持久挂载：
`<REMOTE_MEMORY_DIR>/projects/<sanitized-git-root>/agent-memory-local/<agentType>/`

---

## 6. 会话持久化与恢复

### 6.1 JSONL 转录文件

**文件**: `utils/sessionStorage.ts`

每个会话对应一个 JSONL 文件，存储路径：

```
~/.claude/projects/<project-subdir>/<session-id>.jsonl
```

文件内容为逐行 JSON 条目，包含：
- 完整的 `Message` 对象（序列化格式）
- 文件历史快照 (`FileHistorySnapshotMessage`)
- Attribution 快照 (`AttributionSnapshotMessage`)
- Context Collapse 快照/提交
- Session Metadata（标题、标签）
- Worktree 状态

**清理策略**: `cleanupPeriodDays` 设置（默认 30 天），设为 0 则完全不写入转录。

### 6.2 Session Resume 机制

**文件**: `utils/sessionRestore.ts`, `utils/conversationRecovery.ts`

支持多种恢复模式：

| CLI 参数 | 功能 |
|----------|------|
| `--resume` | 恢复特定会话 |
| `--continue` / `-c` | 继续最近的会话 |
| `--from-pr` | 从 PR 恢复 |
| `--resume-session-at` | 恢复到特定时间点 |
| `--rewind-files` | 回退文件状态 |

恢复流程从 JSONL 转录加载消息、文件历史、attribution 状态、context-collapse 状态、todo 列表等。

**Session 搜索** (`utils/agenticSessionSearch.ts`): 加载磁盘上的 lite/full 日志，使用 LLM sideQuery 进行语义排序（非向量索引）。

### 6.3 Tool Result 溢出存储

**文件**: `utils/toolResultStorage.ts`

当工具输出过大时，内容被持久化到磁盘：

```
<project-dir>/<session-id>/tool-results/
```

消息中的内容被替换为 `<persisted-output>` XML 标签 + 预览片段。恢复会话时自动重新加载。

---

## 7. 查询管线中的记忆处理流水线

**文件**: `query.ts` (主循环)

每次 API 调用前，消息数组经过以下处理管线：

```
Step 1: getMessagesAfterCompactBoundary(messages)
        ↓ 只取最后一个 compact boundary 之后的消息
        
Step 2: applyToolResultBudget(messages)
        ↓ 将超大工具输出溢出到磁盘
        
Step 3: snipProjection(messages) [if HISTORY_SNIP]
        ↓ 为模型裁剪历史视图，计算 snipTokensFreed
        
Step 4: microcompactMessages(messages)
        ↓ 清理/缩小旧工具结果
        
Step 5: contextCollapse(messages) [if CONTEXT_COLLAPSE]
        ↓ 进一步投影/折叠上下文
        
Step 6: autoCompactIfNeeded(messages) 
        ↓ 检查阈值，可能触发压缩（SM-compact 或 full compact）
        ↓ 若压缩成功，messagesForQuery = buildPostCompactMessages(result)
        
Step 7: prependUserContext(messages, userContext)
        ↓ 注入 CLAUDE.md 等项目上下文
        
Step 8: normalizeMessagesForAPI(messages, tools)
        ↓ 过滤/合并/修复消息格式（thinking 合并、tool_use/result 配对等）
        
Step 9: API 调用 (callModel)
```

**系统提示组装**:
```typescript
const fullSystemPrompt = asSystemPrompt(
  appendSystemContext(systemPrompt, systemContext)
)
```
- `systemPrompt`: 静态默认提示或自定义提示
- `systemContext`: git status 等动态键值对
- `userContext`: CLAUDE.md、cwd 等（作为首条 user message 注入）

---

## 8. 压缩后的清理与缓存失效

**文件**: `services/compact/postCompactCleanup.ts`

`runPostCompactCleanup()` 在每次压缩后执行，确保缓存一致性：

```typescript
function runPostCompactCleanup(querySource?: QuerySource): void {
  // 1. 重置微压缩状态
  resetMicrocompactState()
  
  // 2. 重置 Context Collapse（仅主线程）
  if (isMainThreadCompact) resetContextCollapse()
  
  // 3. 清除 getUserContext 缓存（仅主线程）
  if (isMainThreadCompact) {
    getUserContext.cache.clear()
    resetGetMemoryFilesCache('compact')
  }
  
  // 4. 清除系统提示 sections 缓存
  clearSystemPromptSections()
  
  // 5. 清除分类器审批缓存
  clearClassifierApprovals()
  
  // 6. 清除推测性权限检查
  clearSpeculativeChecks()
  
  // 7. 清除 beta tracing 状态
  clearBetaTracingState()
  
  // 8. 清除 session messages 缓存
  clearSessionMessagesCache()
  
  // 9. 清除文件内容缓存（commit attribution）
  sweepFileContentCache()
}
```

**子代理安全**: 子代理 (`agent:*`) 与主线程共享模块级状态。只有主线程压缩才重置 context-collapse 和 memory file 缓存，防止子代理压缩破坏主线程状态。

---

## 9. 子代理记忆管理

**文件**: `services/inProcessRunner.ts`

in-process 子代理（如 swarm agent）拥有独立的消息历史，但使用相同的压缩机制：

- 使用 `tokenCountWithEstimation` 评估上下文大小
- 使用 `getAutoCompactThreshold` 判断是否需要压缩
- 调用 `compactConversation` 执行压缩
- 使用 `buildPostCompactMessages` 重建消息
- 压缩后调用 `resetMicrocompactState`

子代理的 Session Memory 提取只在 `querySource === 'repl_main_thread'` 时运行，子代理不触发独立的 Session Memory 提取。

---

## 10. 配置参考

### 环境变量

| 变量 | 作用 |
|------|------|
| `DISABLE_COMPACT` | 完全禁用压缩（自动 + 手动） |
| `DISABLE_AUTO_COMPACT` | 仅禁用自动压缩（保留手动 `/compact`） |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | 覆盖自动压缩百分比阈值（0-100） |
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW` | 覆盖上下文窗口大小 |
| `CLAUDE_CODE_BLOCKING_LIMIT_OVERRIDE` | 覆盖阻塞限制 |
| `CLAUDE_CODE_MAX_CONTEXT_TOKENS` | 限制最大上下文 token |
| `CLAUDE_CODE_DISABLE_1M_CONTEXT` | 禁用 1M 上下文 |
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY` | 禁用自动记忆 |
| `CLAUDE_CODE_REMOTE_MEMORY_DIR` | 远程模式下的记忆基础目录 |
| `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` | Cowork 全路径覆盖 auto-memory 目录 |
| `CLAUDE_CODE_SIMPLE` / `--bare` | 精简模式（禁用 auto-memory、CLAUDE.md 自动发现等） |
| `ENABLE_CLAUDE_CODE_SM_COMPACT` | 强制启用 SM compact |
| `DISABLE_CLAUDE_CODE_SM_COMPACT` | 强制禁用 SM compact |

### 用户设置

| 设置 | 默认值 | 作用 |
|------|--------|------|
| `autoCompactEnabled` | `true` | 全局自动压缩开关 |
| `autoMemoryEnabled` | - | 自动记忆开关 |
| `autoMemoryCustomDirectory` | - | 自定义记忆目录 |
| `cleanupPeriodDays` | `30` | 转录清理周期（0 = 不写入） |
| `claudeMdExcludes` | - | 排除的 CLAUDE.md 路径 |

### Feature Flags (GrowthBook)

| Flag | 作用 |
|------|------|
| `tengu_session_memory` | 启用 Session Memory 提取 |
| `tengu_sm_config` | Session Memory 配置参数 |
| `tengu_sm_compact` | 启用 Session Memory Compact |
| `tengu_sm_compact_config` | SM Compact 配置参数 |
| `HISTORY_SNIP` | 启用历史裁剪 |
| `REACTIVE_COMPACT` | 启用响应式压缩 |
| `CONTEXT_COLLAPSE` | 启用上下文折叠 |
| `CACHED_MICROCOMPACT` | 启用缓存微压缩 |
| `PROMPT_CACHE_BREAK_DETECTION` | 启用 prompt cache 断裂检测 |
| `tengu_compact_cache_prefix` | 压缩时复用 prompt cache |
| `tengu_passport_quail` | 启用后台记忆提取 (extractMemories) |
| `tengu_bramble_lintel` | 记忆提取频率控制 (每 N 轮, 默认 1) |
| `TEAMMEM` | 启用 Team Memory |
| `EXTRACT_MEMORIES` | 编译时 feature flag: 启用记忆提取代码路径 |

---

## 11. 核心数据结构

### Message 类型

```typescript
type Message = 
  | UserMessage
  | AssistantMessage
  | SystemMessage            // 包括 SystemCompactBoundaryMessage
  | AttachmentMessage
  | HookResultMessage
  | ProgressMessage

interface SystemCompactBoundaryMessage extends SystemMessage {
  subtype: 'compact_boundary'
  compactMetadata: {
    preservedSegment?: {
      headUuid: UUID
      anchorUuid: UUID
      tailUuid: UUID
    }
    preCompactDiscoveredTools?: string[]
  }
}
```

### CompactionResult

```typescript
interface CompactionResult {
  boundaryMarker: SystemMessage          // compact_boundary 标记
  summaryMessages: UserMessage[]          // 摘要消息
  attachments: AttachmentMessage[]        // 恢复附件
  hookResults: HookResultMessage[]        // Hook 结果
  messagesToKeep?: Message[]              // 保留的尾部消息
  userDisplayMessage?: string             // UI 显示消息
  preCompactTokenCount?: number           // 压缩前 token 数
  postCompactTokenCount?: number          // 压缩 API 调用的 token 数
  truePostCompactTokenCount?: number      // 压缩后实际上下文 token 数
  compactionUsage?: Usage                 // API 使用统计
}
```

### SessionMemoryConfig

```typescript
interface SessionMemoryConfig {
  minimumMessageTokensToInit: number      // 默认 10,000
  minimumTokensBetweenUpdate: number      // 默认 5,000
  toolCallsBetweenUpdates: number         // 默认 3
}

interface SessionMemoryCompactConfig {
  minTokens: number                       // 默认 10,000
  minTextBlockMessages: number            // 默认 5
  maxTokens: number                       // 默认 40,000
}
```

### AutoCompactTrackingState

```typescript
interface AutoCompactTrackingState {
  compacted: boolean                      // 本链是否已压缩
  turnCounter: number                     // 轮次计数
  turnId: string                          // 唯一轮次 ID
  consecutiveFailures?: number            // 连续失败次数（熔断器）
}
```

---

## 12. 关键源文件索引

| 领域 | 文件路径 |
|------|----------|
| **主查询循环** | `src/query.ts` |
| **QueryEngine** | `src/QueryEngine.ts` |
| **自动压缩阈值** | `src/services/compact/autoCompact.ts` |
| **完整压缩/摘要** | `src/services/compact/compact.ts` |
| **SM Compact** | `src/services/compact/sessionMemoryCompact.ts` |
| **微压缩** | `src/services/compact/microCompact.ts` |
| **压缩后清理** | `src/services/compact/postCompactCleanup.ts` |
| **Session Memory 提取** | `src/services/SessionMemory/sessionMemory.ts` |
| **SM 工具函数** | `src/services/SessionMemory/sessionMemoryUtils.ts` |
| **SM Prompt** | `src/services/SessionMemory/prompts.ts` |
| **Token 计数** | `src/utils/tokens.ts` |
| **Token 粗估** | `src/services/tokenEstimation.ts` |
| **消息处理/Boundary** | `src/utils/messages.ts` |
| **CLAUDE.md 加载** | `src/utils/claudemd.ts` |
| **用户上下文** | `src/utils/queryContext.ts` |
| **API 归一化** | `src/services/api/claude.ts` |
| **User/System 上下文注入** | `src/utils/api.ts` |
| **会话存储** | `src/utils/sessionStorage.ts` |
| **会话恢复** | `src/utils/sessionRestore.ts` |
| **Tool Result 溢出** | `src/utils/toolResultStorage.ts` |
| **项目上下文** | `src/utils/context.ts` |
| **Auto Memory 路径/开关** | `src/memdir/paths.ts` |
| **Auto Memory 主逻辑** | `src/memdir/memdir.ts` |
| **记忆类型定义** | `src/memdir/memoryTypes.ts` |
| **后台记忆提取** | `src/services/extractMemories/extractMemories.ts` |
| **提取 Prompt** | `src/services/extractMemories/prompts.ts` |
| **Team Memory 路径** | `src/memdir/teamMemPaths.ts` |
| **Team Memory 同步** | `src/services/teamMemorySync/index.ts` |
| **Team Memory 安全** | `src/services/teamMemorySync/teamMemSecretGuard.ts` |
| **Agent Memory** | `src/tools/AgentTool/agentMemory.ts` |
| **项目 Onboarding** | `src/projectOnboardingState.ts` |
| **全局配置** | `src/utils/config.ts` |
