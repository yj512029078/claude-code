---
title: Claude Code 与 LLM 交互深度技术文档
aliases: [LLM 交互, LLM Interaction]
series: Claude Code 架构解析
category: 核心运行时
order: 4
tags:
  - claude-code
  - llm
  - api
  - streaming
  - prompt-caching
  - cost-tracking
date: 2026-04-04
---

# Claude Code 与 LLM 交互深度技术文档

## 1. 概览

Claude Code 与 LLM 的交互是一个**多层管道**，从模型选择、请求构建、流式通信、工具执行到成本追踪，形成完整的闭环。

```
┌─────────────────────────────────────────────────────────────┐
│                    query() — Agentic 循环                    │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 消息准备 → 压缩检查 → System Prompt → 工具 Schema     │   │
│  └──────────────────────────┬───────────────────────────┘   │
│                             ▼                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ queryModelWithStreaming() — API 调用层                 │   │
│  │  ├── 模型选择 (model.ts / providers.ts)               │   │
│  │  ├── Beta 特性 (betas.ts)                            │   │
│  │  ├── Thinking 配置 (thinking.ts)                     │   │
│  │  ├── Effort 配置 (effort.ts)                         │   │
│  │  ├── Prompt 缓存 (addCacheBreakpoints)               │   │
│  │  └── 重试逻辑 (withRetry.ts)                         │   │
│  └──────────────────────────┬───────────────────────────┘   │
│                             ▼                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Anthropic SDK — HTTP/SSE 流式请求                     │   │
│  │  beta.messages.create({ ...params, stream: true })    │   │
│  └──────────────────────────┬───────────────────────────┘   │
│                             ▼                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 流式处理 + 工具执行                                    │   │
│  │  ├── StreamingToolExecutor (流中执行工具)              │   │
│  │  ├── runTools (流后批量执行)                           │   │
│  │  └── 结果注入消息 → 下一轮 API 调用                    │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. API 客户端工厂 (`services/api/client.ts`)

### 2.1 四种 Provider 自动选择

```typescript
getAnthropicClient()
    │
    ├── CLAUDE_CODE_USE_BEDROCK → AnthropicBedrock SDK
    │   └── AWS Region + Bearer Token / AWS Credentials
    │
    ├── CLAUDE_CODE_USE_VERTEX → AnthropicVertex SDK  
    │   └── Google Auth + Project + Region
    │
    ├── CLAUDE_CODE_USE_FOUNDRY → AnthropicFoundry SDK
    │   └── API Key 或 Azure AD Token Provider
    │
    └── 默认 → Anthropic SDK (第一方)
        ├── API Key 模式: ANTHROPIC_API_KEY
        └── OAuth 模式: Claude AI 订阅 Token (自动刷新)
```

### 2.2 客户端共享配置

```typescript
const ARGS = {
  maxRetries: 0,                    // SDK 层不重试，由 withRetry 管理
  timeout: API_TIMEOUT_MS,          // 默认 600 秒
  dangerouslyAllowBrowser: true,    // CLI 环境
  defaultHeaders: {
    'x-app': 'claude-code',
    'User-Agent': userAgent,
    'x-session-id': sessionId,
    // 可选: x-container-id, x-client-app, 自定义头
  },
  fetchOptions: proxyConfig,        // 代理配置
}
```

### 2.3 请求追踪

```typescript
// 第一方 API 专用: 每个请求注入唯一 ID
buildFetch(originalFetch) → wrappedFetch
  // 每次 fetch 时注入:
  headers['x-client-request-id'] = randomUUID()
  // 用于: 问题排查、日志关联
```

---

## 3. 模型选择 (`utils/model/`)

### 3.1 选择优先级

```
用户指定 (最高优先级)
  ├── 会话内 /model 命令覆盖
  ├── CLI --model 参数
  ├── ANTHROPIC_MODEL 环境变量
  └── settings.json 中的 model 设置

默认值 (无用户指定时)
  ├── 订阅类型决定:
  │   ├── Max/Team Premium → Opus 4.6 [1m]
  │   ├── 普通订阅 → Sonnet 4.6
  │   └── API Key → Sonnet 4.6
  └── Provider 影响默认:
      ├── firstParty → Sonnet 4.6
      └── Bedrock/Vertex → Sonnet 4.5 (兼容性)

运行时调整
  ├── Plan 模式: haiku → Sonnet, opusplan → Opus
  └── Fast 模式: 强制使用特定快速模型
```

### 3.2 模型字符串映射

```typescript
// 同一个逻辑模型在不同 Provider 有不同 ID
getModelStrings()
  firstParty: "claude-sonnet-4-6-20261022"
  bedrock:    "us.anthropic.claude-sonnet-4-6-20261022-v1:0"
  vertex:     "claude-sonnet-4-6@20261022"
  foundry:    "claude-sonnet-4-6-20261022"

// 用户可以通过 modelOverrides 自定义映射
settings.modelOverrides: { "sonnet46": "my-custom-arn" }
```

### 3.3 上下文窗口

```typescript
getContextWindowForModel(model):
  // 默认: 200,000 tokens
  // [1m] 后缀: 1,000,000 tokens
  // 特殊模型有专属窗口大小
  // Ant 内部可通过 CLAUDE_CODE_MAX_CONTEXT_TOKENS 覆盖
```

---

## 4. 请求构建 (`services/api/claude.ts`)

### 4.1 完整请求参数

```typescript
// queryModel 内部构建的请求参数
const params: BetaMessageStreamParams = {
  model: normalizeModelStringForAPI(currentModel),
  max_tokens: getMaxOutputTokensForModel(model),
  messages: normalizedMessages,        // 规范化后的对话消息
  system: systemBlocks,                // 系统提示词块 (带缓存控制)
  tools: toolSchemas,                  // 工具 JSON Schema
  metadata: { user_id: hashedUserId }, // 用户标识 (哈希)
  
  // 可选:
  thinking: thinkingConfig,            // 思考配置
  output_config: { effort },           // 努力等级
  betas: betaHeaders,                  // Beta 特性
  // ...extra body params
}

// 实际 HTTP 调用:
const result = await anthropic.beta.messages
  .create({ ...params, stream: true }, { signal, headers })
  .withResponse()  // 获取 request_id 和 Response
```

### 4.2 消息规范化

```
原始内部消息 (Message[])
    │
    ▼
normalizeMessagesForAPI()
    │
    ├── 过滤系统消息 (不发送给 API)
    ├── 合并相邻的同角色消息
    ├── 确保 user/assistant 交替
    ├── 处理附件消息
    ├── tool_use / tool_result 配对检查
    └── 图片/PDF 内容处理
    │
    ▼
API 格式消息 (MessageParam[])
```

### 4.3 工具 Schema 构建

```typescript
toolToAPISchema(tool): BetaToolWithExtras
    │
    ├── name: tool.name
    ├── description: await tool.description()  // 动态描述
    ├── input_schema: tool.inputJSONSchema     // Zod → JSON Schema
    ├── cache_control: getCacheControl(...)     // 缓存控制
    ├── defer_loading: tool.shouldDefer         // 延迟加载标记
    └── eager_input_streaming: true (可选)      // 流式输入

// Session 级缓存: 同一工具不重复构建 Schema
// 缓存 Key: tool.name (内建) 或 name+schema (MCP/动态)
```

---

## 5. Thinking & Effort 配置

### 5.1 Thinking (扩展思考)

```typescript
ThinkingConfig = 
  | { type: 'adaptive' }     // 自适应 (模型自决定是否思考)
  | { type: 'enabled', budgetTokens: number }  // 强制启用 + token 预算
  | { type: 'disabled' }     // 禁用

// 决策逻辑:
if (CLAUDE_CODE_DISABLE_THINKING) → disabled
if (modelSupportsAdaptiveThinking(model)) → adaptive
if (modelSupportsThinking(model)) → enabled + budgetTokens
else → disabled

// budgetTokens 计算:
// 默认: getMaxThinkingTokensForModel(model)
// 用户可通过 MAX_THINKING_TOKENS 环境变量覆盖
// 硬约束: budgetTokens < maxOutputTokens - 1
```

### 5.2 Effort (努力等级)

```typescript
type EffortLevel = 'low' | 'medium' | 'high' | 'max'

// 生效方式:
if (modelSupportsEffort(model)) {
  // 字符串 effort → API output_config.effort
  params.output_config = { effort: 'high' }
  
  // 数字 effort (内部) → anthropic_internal.effort_override
  // 仅 Ant 用户
}

// 'max' effort 限制:
// 仅 opus-4-6 和特定内部模型支持
// 不支持时自动降级为 'high'
```

---

## 6. Prompt 缓存系统

### 6.1 系统提示词缓存

```typescript
buildSystemPromptBlocks(systemPrompt, options):
    │
    ├── 将系统提示词拆分为多个文本块
    ├── 每个块标记 cacheScope:
    │   ├── 'global' — 跨用户可缓存 (工具定义等)
    │   ├── 'session' — 会话级缓存 (上下文特定)
    │   └── null — 不缓存
    └── 生成带 cache_control 的 API system blocks

// API 限制: 不能添加太多 cache_control 块，否则返回 400
```

### 6.2 消息级缓存

```typescript
addCacheBreakpoints(messages, options):
    │
    ├── 在最后一条 user/assistant 消息上加 cache_control
    │   // 这样模型可以缓存这条消息之前的所有内容
    │
    ├── skipCacheWrite 场景:
    │   // fork Agent / 无操作合并时
    │   // 在倒数第二条消息加标记 (不缓存最后一条)
    │
    └── 可选: cache_edits 和 cache_reference
        // 用于增量编辑缓存内容
```

### 6.3 工具排序与缓存稳定性

```
工具列表排序策略:
  1. 内建工具: 按名称排序 → 稳定前缀
  2. MCP 工具: 按名称排序 → 独立分区
  3. uniqBy('name') 去重 → 内建优先

目的: 即使 MCP 工具变化，内建工具的缓存前缀不受影响
→ 最大化 prompt 缓存命中率
```

---

## 7. Beta 特性管理 (`utils/betas.ts`)

```typescript
getMergedBetas(model, options): string[]
    │
    ├── 基础 Beta:
    │   ├── 'claude-code-2025-04-04'    // Claude Code 专属
    │   ├── 'interleaved-thinking-2025-05-14'  // 交错思考
    │   └── 'prompt-caching-2024-07-31' // Prompt 缓存
    │
    ├── 条件 Beta:
    │   ├── 1M 上下文 → 'long-context-1m'
    │   ├── OAuth → 'oauth-2025-04-18'
    │   ├── Redacted Thinking → 'redacted-thinking'
    │   ├── 结构化输出 → 'structured-outputs'
    │   ├── Tool Search → 'tool-search'
    │   ├── Web Search (Vertex) → 'vertex-web-search'
    │   └── Context Management → 'context-management'
    │
    ├── 环境变量追加:
    │   └── ANTHROPIC_BETAS → 额外 beta 标签
    │
    └── SDK Beta 过滤:
        // API Key 用户只允许白名单内的 SDK beta
        // 订阅用户的自定义 beta 被忽略
```

---

## 8. 流式交互详细流程

### 8.1 完整数据流

```
[1. 请求发出]
query() → queryLoop() → deps.callModel()
    │
    ▼
queryModelWithStreaming() → queryModel()
    │
    ├── 构建 params (messages, system, tools, betas...)
    ├── withRetry 包裹 (自动重试)
    └── anthropic.beta.messages.create({ stream: true })
    │
    ▼
[2. SSE 流式响应]
SDK 返回 AsyncIterable<StreamEvent>
    │
    ├── message_start: { message: { id, model, usage } }
    ├── content_block_start: { content_block: { type: 'text' | 'tool_use' | 'thinking' } }
    ├── content_block_delta: { delta: { text | partial_json | thinking } }
    ├── content_block_stop: { }
    ├── message_delta: { delta: { stop_reason }, usage: { output_tokens } }
    └── message_stop: { }
    │
    ▼
[3. 流处理 (query.ts)]
for await (const event of stream):
    │
    ├── 文本块 → yield 给 UI (用户实时看到)
    │
    ├── 思考块 → yield 给 UI (可选显示)
    │
    ├── tool_use 块 → 两条路径:
    │   │
    │   ├── StreamingToolExecutor 启用:
    │   │   └── executor.addTool(block)
    │   │       → 立即开始执行 (流中并行)
    │   │       → 完成的结果通过 getCompletedResults() 获取
    │   │
    │   └── StreamingToolExecutor 未启用:
    │       └── 收集到 toolUseBlocks[]
    │           → 流结束后 runTools() 批量执行
    │
    ├── 用量信息 → updateUsage() → cost-tracker
    │
    └── 停止原因 → 决定是否需要续轮
    │
    ▼
[4. 工具执行]
    │
    ├── StreamingToolExecutor.getRemainingResults()
    │   └── 等待所有流中启动的工具完成
    │
    └── runTools() (非流式路径)
        ├── partitionToolCalls(): 分为串行/并行批次
        ├── 并行安全的工具 → Promise.all 并行执行
        ├── 非并行安全的工具 → 逐个串行执行
        └── 每个工具: runToolUse() → tool.call()
    │
    ▼
[5. 结果注入]
    │
    ├── 工具结果构建为 tool_result 消息
    ├── applyToolResultBudget(): 裁剪过大结果
    ├── 追加到 messages[]
    └── needsFollowUp = true → 回到步骤 1
    │
    ▼
[6. 循环终止条件]
    │
    ├── stop_reason === 'end_turn' (模型完成)
    ├── stop_reason === 'refusal' (拒绝)
    ├── maxTurns 达到上限
    ├── token budget 耗尽
    ├── abortController.signal.aborted (用户取消)
    └── 无更多工具调用
```

### 8.2 StreamingToolExecutor — 流中工具执行

```
模型输出: "让我搜索代码..." [tool_use: Grep] [tool_use: Glob]
           ↓ 流式接收中...     ↓ 立即启动      ↓ 立即启动

时间轴:
t0: 收到 tool_use Grep  → addTool → 开始执行 Grep
t1: 收到 tool_use Glob  → addTool → 开始执行 Glob (并行)
t2: Grep 完成           → 结果就绪
t3: 模型输出完成         → 流结束
t4: Glob 完成           → 结果就绪
t5: getRemainingResults() → 收集所有结果

对比非流式:
t0-t3: 等待模型完整输出
t3: 开始执行 Grep
t4: Grep 完成，开始执行 Glob (或并行)
t5: Glob 完成

→ 流式工具执行节省了 t0-t3 的等待时间
```

### 8.3 工具并发控制

```typescript
// StreamingToolExecutor 的并发模型:
tool.isConcurrencySafe()
  true  → 可以与其他工具并行执行 (Grep, Glob, FileRead, WebFetch...)
  false → 必须独占执行 (Bash, FileEdit, FileWrite...)

// 执行队列:
processQueue():
  if (正在执行非并发工具) → 等待
  if (队列中有非并发工具) → 等待所有并发工具完成 → 独占执行
  else → 所有并发安全工具并行启动

// 子 AbortController:
// Bash 失败时可以中止同批次的其他子进程
// 但不中止整个查询
```

---

## 9. 重试与错误处理

### 9.1 重试策略 (`withRetry.ts`)

```
withRetry(operation, options) — AsyncGenerator
    │
    ├── 可重试错误:
    │   ├── 429 Too Many Requests (有条件)
    │   ├── 529 Overloaded (限制次数 + 可选 fallback 模型)
    │   ├── 408 Timeout
    │   ├── 409 Conflict
    │   ├── 5xx Server Error
    │   ├── 401 Unauthorized (清除缓存后重试)
    │   ├── ECONNRESET / EPIPE (连接错误)
    │   └── OAuth Token Revoked (刷新后重试)
    │
    ├── 不可重试错误:
    │   ├── 400 Bad Request
    │   ├── 403 Forbidden (非 token 问题)
    │   ├── 413 Payload Too Large (Prompt Too Long)
    │   └── 模型不支持的特性
    │
    ├── 重试延迟:
    │   └── 指数退避 + 抖动 + Retry-After 头
    │
    ├── 529 特殊处理:
    │   ├── MAX_529_RETRIES 限制
    │   ├── 超限 → FallbackTriggeredError → 切换模型
    │   └── 后台查询可跳过重试 (避免放大)
    │
    └── 持久模式 (CLAUDE_CODE_UNATTENDED_RETRY):
        └── 429/529 时分段长时间等待
```

### 9.2 错误映射 (`errors.ts`)

```
SDK/网络错误 → getAssistantMessageFromError() → 用户可见消息

映射示例:
  APIConnectionTimeoutError → "请求超时，请重试"
  429 + rate-limit headers  → "达到速率限制" + 等待时间
  Prompt Too Long           → "上下文过长" + 建议 /compact
  401 (bad API key)         → "API Key 无效" + 设置指引
  OAuth Revoked             → "登录已过期" + /login 指引
  stopReason === 'refusal'  → "模型拒绝回答"
```

---

## 10. 成本追踪 (`cost-tracker.ts`)

```typescript
// 每次 API 响应后:
addToTotalSessionCost(cost, usage, model)
    │
    ├── 更新总成本: addToTotalCostState(cost)
    ├── 按模型聚合:
    │   addToTotalModelUsage(model, {
    │     inputTokens, outputTokens,
    │     cacheCreationTokens, cacheReadTokens,
    │     webSearchRequests,
    │     costUSD: calculateUSDCost(model, usage)
    │   })
    ├── OpenTelemetry 指标:
    │   ├── getCostCounter().add(cost)
    │   └── getTokenCounter().add(tokens, { model, direction })
    └── Advisor 用量 (如果启用):
        └── getAdvisorUsage() → 递归追加

// 会话持久化:
saveCurrentSessionCosts()     // 保存到项目配置
restoreCostStateForSession()  // 恢复历史会话成本
```

---

## 11. 非主循环的 LLM 调用

### 11.1 queryHaiku — 快速小模型

```typescript
queryHaiku({ systemPrompt, userPrompt, outputFormat, signal, options })
    │
    ├── 使用 getSmallFastModel() (通常是 Haiku)
    ├── 无工具 (tools: [])
    ├── 禁用 Thinking
    ├── 可选结构化输出 (JSON Schema)
    └── 非流式消费 (内部仍是 stream: true)

用途:
  - Bash 命令安全分类器
  - 权限自动判断 (YOLO classifier)
  - 工具结果摘要生成
  - 其他需要快速/廉价推理的场景
```

### 11.2 queryWithModel — 指定模型

```typescript
// 用于需要特定模型的场景 (如 ULTRAPLAN 用 Opus 4.6)
queryWithModel({ model, messages, systemPrompt, ... })
```

### 11.3 executeNonStreamingRequest — 非流式回退

```typescript
// 当不需要流式或作为 fallback:
anthropic.beta.messages.create(
  { ...params },  // 注意: 没有 stream: true
  { signal, timeout: fallbackTimeoutMs }
)
```

---

## 12. 完整一轮交互的时序图

```
用户输入: "找到所有 TODO 并修复它们"

[t0] query() 开始新的一轮
    │
[t1] 消息准备:
    │  - 上下文压缩检查 (token 数 > 阈值?)
    │  - system prompt 构建 (工具列表 + 记忆 + 自定义指令)
    │  - 消息规范化 (normalizeMessagesForAPI)
    │  - 缓存断点设置 (addCacheBreakpoints)
    │
[t2] HTTP 请求发出:
    │  POST /v1/messages (stream: true)
    │  Headers: anthropic-beta, x-client-request-id, ...
    │  Body: { model, messages, system, tools, thinking, ... }
    │
[t3] SSE 流开始:
    │  ← message_start (model, usage)
    │  ← content_block_start (thinking)
    │  ← content_block_delta ("让我先搜索TODO...")  → 用户看到文本
    │  ← content_block_stop
    │  ← content_block_start (text)
    │  ← content_block_delta ("我来搜索...")  → 用户看到文本
    │  ← content_block_stop
    │  ← content_block_start (tool_use: Grep)
    │  ← content_block_delta ({"pattern": "TODO"})
    │
[t4] StreamingToolExecutor 启动 Grep:
    │  → Grep 开始执行 (流还在继续)
    │
    │  ← content_block_stop (Grep 参数完整)
    │  ← content_block_start (tool_use: Glob)
    │  ← content_block_delta ({"pattern": "**/*.ts"})
    │
[t5] StreamingToolExecutor 启动 Glob:
    │  → Glob 开始执行 (与 Grep 并行)
    │
    │  ← message_delta (stop_reason: "tool_use")
    │  ← message_stop
    │
[t6] 流结束，等待工具完成:
    │  Grep 完成: 47 matches
    │  Glob 完成: 156 files
    │
[t7] 工具结果注入消息:
    │  messages += [assistantMessage, toolResult_Grep, toolResult_Glob]
    │  needsFollowUp = true
    │
[t8] 回到 t1，开始第二轮:
    │  → 模型看到搜索结果，决定编辑文件
    │  → tool_use: FileEdit × 5
    │  → ...循环直到模型完成
    │
[t_final] stop_reason === 'end_turn'
    │  → 返回最终结果给用户
    │  → 更新成本追踪
    │  → 记录转录
```

---

## 13. 关键设计决策

### 13.1 为什么始终使用 stream: true

即使 `queryModelWithoutStreaming` 也内部使用流式请求——这保证了：
- 统一的代码路径（减少分支）
- 长时间请求不会超时（流式保持连接活跃）
- 可以提前获取 request_id 用于排障

### 13.2 为什么流中执行工具

StreamingToolExecutor 在**模型还在输出时**就开始执行工具：
- 多工具场景节省数秒到数十秒
- 第一个工具的执行时间被后续工具的参数流覆盖
- 并发安全工具（Grep、Glob、FileRead）可以完全并行

### 13.3 为什么自己管理重试而非用 SDK

`withRetry` 是 AsyncGenerator，可以在重试间**yield 状态消息**给 UI：
- 用户看到 "正在重试 (2/3)..." 而非假死
- 529 过载时可以**切换 fallback 模型**
- 401 时可以**刷新 OAuth token** 后重试
- 429 时可以根据 `Retry-After` 精确等待

### 13.4 为什么 Prompt 缓存如此复杂

Anthropic API 的 Prompt 缓存按**前缀匹配**——只有请求的前缀部分相同时才命中缓存。因此：
- 系统提示词按 cacheScope 分块，全局部分（工具定义）放前面
- 工具按名称排序确保顺序稳定
- 内建工具和 MCP 工具分区，避免 MCP 变化破坏内建缓存
- 消息级缓存断点精确控制缓存边界
