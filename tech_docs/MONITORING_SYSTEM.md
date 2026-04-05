---
title: Claude Code 全链路监控系统
aliases: [监控系统, Monitoring System]
series: Claude Code 架构解析
category: UI与可观测性
order: 19
tags:
  - claude-code
  - monitoring
  - sentry
  - datadog
  - feature-flags
  - observability
date: 2026-04-04
---

# Claude Code 全链路监控系统 — 深度技术文档

## 目录

1. [架构概览](#1-架构概览)
2. [分布式追踪层 (Session Tracing)](#2-分布式追踪层-session-tracing)
   - 2.1 [Span 层级模型](#21-span-层级模型)
   - 2.2 [AsyncLocalStorage 上下文传播](#22-asynclocalstorage-上下文传播)
   - 2.3 [Span 生命周期管理](#23-span-生命周期管理)
   - 2.4 [Beta 增强追踪](#24-beta-增强追踪)
3. [Perfetto 本地追踪层](#3-perfetto-本地追踪层)
   - 3.1 [Chrome Trace Event 格式](#31-chrome-trace-event-格式)
   - 3.2 [多 Agent 进程映射](#32-多-agent-进程映射)
   - 3.3 [可视化工作流](#33-可视化工作流)
4. [OpenTelemetry 基础设施](#4-opentelemetry-基础设施)
   - 4.1 [三信号初始化](#41-三信号初始化)
   - 4.2 [导出器配置](#42-导出器配置)
   - 4.3 [BigQuery 自定义导出](#43-bigquery-自定义导出)
   - 4.4 [延迟初始化与关闭](#44-延迟初始化与关闭)
5. [指标聚合层 (Metrics)](#5-指标聚合层-metrics)
   - 5.1 [全局状态计数器](#51-全局状态计数器)
   - 5.2 [OTel Meter 指标](#52-otel-meter-指标)
   - 5.3 [API Turn 指标消息](#53-api-turn-指标消息)
6. [事件分析层 (Analytics)](#6-事件分析层-analytics)
   - 6.1 [零依赖公共 API](#61-零依赖公共-api)
   - 6.2 [Sink 路由架构](#62-sink-路由架构)
   - 6.3 [Datadog 通道](#63-datadog-通道)
   - 6.4 [第一方事件管道](#64-第一方事件管道)
   - 6.5 [采样与门控](#65-采样与门控)
   - 6.6 [PII 防护机制](#66-pii-防护机制)
7. [多级日志层 (Logging)](#7-多级日志层-logging)
   - 7.1 [Debug 日志](#71-debug-日志)
   - 7.2 [诊断日志 (Diagnostics)](#72-诊断日志-diagnostics)
   - 7.3 [错误日志](#73-错误日志)
   - 7.4 [OTel 事件日志](#74-otel-事件日志)
8. [性能剖析 (Profiling)](#8-性能剖析-profiling)
   - 8.1 [启动剖析器](#81-启动剖析器)
   - 8.2 [Headless 剖析器](#82-headless-剖析器)
   - 8.3 [Query 剖析器](#83-query-剖析器)
   - 8.4 [慢操作检测](#84-慢操作检测)
9. [健康保活与异常退出](#9-健康保活与异常退出)
   - 9.1 [远程会话 Heartbeat](#91-远程会话-heartbeat)
   - 9.2 [本地会话活动追踪](#92-本地会话活动追踪)
   - 9.3 [异常退出处理](#93-异常退出处理)
10. [隐私与开关控制](#10-隐私与开关控制)
11. [全链路数据流](#11-全链路数据流)
12. [设计模式总结](#12-设计模式总结)

---

## 1. 架构概览

Claude Code 构建了一套**多层级、多通道**的全链路监控体系，覆盖从用户输入到 LLM 响应、工具执行、再到结果输出的完整生命周期。

```
┌────────────────────────────────────────────────────────────────────────┐
│                       全链路监控架构                                    │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                 │
│  │  OTel Traces  │  │   Perfetto   │  │  OTel Logs   │  ← 追踪层      │
│  │  (Span 树)    │  │  (本地深度)   │  │  (事件日志)   │                │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘                 │
│         │                 │                  │                          │
│  ┌──────┴─────────────────┴──────────────────┴───────┐                │
│  │              OpenTelemetry 基础设施                  │  ← 传输层     │
│  │  MeterProvider · LoggerProvider · TracerProvider    │               │
│  └──────┬──────────────┬─────────────────┬───────────┘                │
│         │              │                 │                              │
│  ┌──────┴───────┐ ┌────┴─────┐  ┌───────┴──────┐                     │
│  │  BigQuery    │ │  OTLP    │  │  Prometheus  │    ← 导出层          │
│  │  (内部 API)  │ │ gRPC/HTTP│  │   (Pull)     │                      │
│  └──────────────┘ └──────────┘  └──────────────┘                      │
│                                                                        │
│  ┌──────────────────────────────────────────────────┐                 │
│  │           事件分析 (Analytics)                     │  ← 事件层      │
│  │  logEvent() → Sink → Datadog + 1P Event Pipeline │                 │
│  └──────────────────────────────────────────────────┘                 │
│                                                                        │
│  ┌─────────────┐ ┌─────────────┐ ┌──────────────┐                    │
│  │ Debug Log   │ │ Diagnostics │ │  Error Log   │   ← 日志层          │
│  │ (文件/stderr)│ │ (JSONL/NoPII)│ │(环形缓冲+JSONL)│                   │
│  └─────────────┘ └─────────────┘ └──────────────┘                    │
│                                                                        │
│  ┌──────────────────────────────────────────────────┐                 │
│  │  Startup · Headless · Query · Slow Ops Profiler  │  ← 剖析层       │
│  └──────────────────────────────────────────────────┘                 │
│                                                                        │
│  ┌──────────────────────────────────────────────────┐                 │
│  │  Heartbeat · Session Activity · Graceful Shutdown │ ← 健康层        │
│  └──────────────────────────────────────────────────┘                 │
└────────────────────────────────────────────────────────────────────────┘
```

核心源文件分布：

| 模块 | 路径 | 职责 |
|------|------|------|
| Session Tracing | `src/utils/telemetry/sessionTracing.ts` | OTel Span 树追踪 |
| Perfetto Tracing | `src/utils/telemetry/perfettoTracing.ts` | Chrome Trace Event 本地追踪 |
| OTel 初始化 | `src/utils/telemetry/instrumentation.ts` | 三信号 Provider 初始化 |
| OTel 事件 | `src/utils/telemetry/events.ts` | `logOTelEvent` 事件发射 |
| BigQuery 导出 | `src/utils/telemetry/bigqueryExporter.ts` | 自定义 Metric Exporter |
| Analytics 公共 API | `src/services/analytics/index.ts` | `logEvent` 队列 + Sink |
| Analytics Sink | `src/services/analytics/sink.ts` | Datadog + 1P 路由 |
| Datadog | `src/services/analytics/datadog.ts` | HTTP Logs Intake |
| Debug 日志 | `src/utils/debug.ts` | 分级文件日志 |
| 诊断日志 | `src/utils/diagLogs.ts` | 容器内 JSONL 无 PII 日志 |
| 错误日志 | `src/utils/log.ts` + `errorLogSink.ts` | 错误持久化 |
| 启动剖析 | `src/utils/startupProfiler.ts` | 启动阶段计时 |
| API 日志 | `src/services/api/logging.ts` | API 调用遥测 |
| 全局状态 | `src/bootstrap/state.ts` | Meter/Counter/全局指标 |
| 优雅退出 | `src/utils/gracefulShutdown.ts` | 异常捕获 + 清理 |

---

## 2. 分布式追踪层 (Session Tracing)

### 来源

`src/utils/telemetry/sessionTracing.ts` (~928 行)

### 2.1 Span 层级模型

Session Tracing 是全链路监控的**主干**。基于 OpenTelemetry Span 模型，定义了 6 种 Span 类型，构成完整的调用链：

```
interaction (根 span — 一次用户交互的完整生命周期)
│
├── llm_request (LLM API 调用)
│   └── 属性: model, prompt_length, usage, TTFT, duration_ms
│
├── tool (工具调用)
│   ├── tool.blocked_on_user (等待用户权限确认)
│   └── tool.execution (工具实际执行)
│
└── hook (钩子执行)
```

类型定义：

```typescript
type SpanType =
  | 'interaction'
  | 'llm_request'
  | 'tool'
  | 'tool.blocked_on_user'
  | 'tool.execution'
  | 'hook'

interface SpanContext {
  span: Span
  startTime: number
  attributes: Record<string, string | number | boolean>
  ended?: boolean
  perfettoSpanId?: string   // 关联 Perfetto 追踪
}
```

每个 Span 携带的**基础属性**由 `getTelemetryAttributes()` 统一注入，包含 session ID、platform、subscription type 等维度信息，确保所有 Span 可在后端按维度聚合。

### 2.2 AsyncLocalStorage 上下文传播

使用两个独立的 `AsyncLocalStorage` 实例分别传播交互级和工具级上下文，在整个异步调用链中**自动传递** Span，无需手动层层穿透：

```typescript
const interactionContext = new AsyncLocalStorage<SpanContext | undefined>()
const toolContext = new AsyncLocalStorage<SpanContext | undefined>()
```

工作流程：

```
startInteractionSpan()
  → interactionContext.enterWith(spanContext)
    → 后续所有 async 函数自动继承
      → startLLMRequestSpan() 读取 interactionContext
      → startToolSpan() 读取 interactionContext
        → toolContext.enterWith(toolSpanContext)
          → startToolExecutionSpan() 读取 toolContext
```

**关键设计**：Span 结束时使用 `enterWith(undefined)` 而非 `exit()`，因为 `exit()` 仅在回调内生效，无法阻止已创建的异步继续体（timers、promise callbacks）继承已结束的 Span 引用。

### 2.3 Span 生命周期管理

**WeakRef + 强引用分离** — 一个精巧的内存管理策略：

```typescript
const activeSpans = new Map<string, WeakRef<SpanContext>>()
const strongSpans = new Map<string, SpanContext>()
```

| Span 类型 | 存储位置 | 理由 |
|-----------|---------|------|
| interaction | ALS (强) + activeSpans (WeakRef) | ALS 持有强引用，span 活跃时不会被 GC |
| tool | ALS (强) + activeSpans (WeakRef) | 同上 |
| llm_request | strongSpans (强) + activeSpans (WeakRef) | 不在 ALS 中存储，需额外强引用防止 GC |
| blocked_on_user | strongSpans (强) | 等待时间可能很长 |
| tool.execution | strongSpans (强) | 执行期间需要保持引用 |
| hook | strongSpans (强) | 钩子执行期间需要保持引用 |

**30 分钟 TTL 安全网** — 后台定时器自动清理孤儿 Span：

```typescript
const SPAN_TTL_MS = 30 * 60 * 1000

function ensureCleanupInterval(): void {
  if (_cleanupIntervalStarted) return
  _cleanupIntervalStarted = true
  const interval = setInterval(() => {
    const cutoff = Date.now() - SPAN_TTL_MS
    for (const [spanId, weakRef] of activeSpans) {
      const ctx = weakRef.deref()
      if (ctx === undefined) {
        // WeakRef 已被 GC 回收
        activeSpans.delete(spanId)
        strongSpans.delete(spanId)
      } else if (ctx.startTime < cutoff) {
        // 超时强制结束
        if (!ctx.ended) ctx.span.end()
        activeSpans.delete(spanId)
        strongSpans.delete(spanId)
      }
    }
  }, 60_000)
  interval.unref() // 不阻塞进程退出
}
```

关键细节：
- **延迟启动** — 在第一次 `startInteractionSpan()` 时才创建定时器，避免从未使用追踪的进程付出开销
- **`unref()`** — 定时器不阻止 Node.js 进程正常退出
- **双重检测** — 先检查 WeakRef 是否已失效（GC 已回收），再检查时间戳

### 2.4 Beta 增强追踪

来源：`src/utils/telemetry/betaSessionTracing.ts`

在标准追踪之上叠加更丰富的属性，主要面向 Honeycomb 等分析后端：

- **内容截断** — `truncateContent()` 对 LLM 输入输出截断，符合 Honeycomb 属性大小限制
- **哈希字段** — 对敏感内容做哈希处理后作为属性存储
- **LLM 请求上下文** — `addBetaLLMRequestAttributes` 添加 new_context（是否新对话）等维度
- **工具输入/输出** — `addBetaToolInputAttributes` / `addBetaToolResultAttributes` 截断后附加

启用条件三级优先级：

```
环境变量 override > ant 内部构建 > GrowthBook feature gate
```

---

## 3. Perfetto 本地追踪层

### 来源

`src/utils/telemetry/perfettoTracing.ts` (~1121 行)

### 3.1 Chrome Trace Event 格式

与 OTel Span **并行运行**（非替代关系），生成可在 [ui.perfetto.dev](https://ui.perfetto.dev) 或 `chrome://tracing` 中可视化的本地 trace 文件：

```typescript
export type TraceEventPhase =
  | 'B'  // Begin duration event
  | 'E'  // End duration event
  | 'X'  // Complete event (with duration)
  | 'i'  // Instant event
  | 'C'  // Counter event
  | 'b'  // Async begin
  | 'n'  // Async instant
  | 'e'  // Async end
  | 'M'  // Metadata event

export type TraceEvent = {
  name: string
  cat: string
  ph: TraceEventPhase
  ts: number          // 时间戳 (微秒)
  pid: number          // 进程 ID
  tid: number          // 线程 ID
  dur?: number         // 持续时间 (微秒)
  args?: Record<string, unknown>
  id?: string          // 异步事件关联 ID
}
```

追踪内容：

| 事件类型 | 记录内容 |
|---------|---------|
| API 请求 | TTFT, TTLT, prompt length, cache stats, message ID, speculative flag |
| 工具执行 | 工具名, 持续时间, token usage |
| 用户输入等待 | 从请求到用户回复的等待时间 |
| Agent 层级 | parent-child 关系（Swarm 多 Agent 场景） |

### 3.2 多 Agent 进程映射

每个 Agent 映射为 Perfetto 中独立的 process + thread，完整还原多 Agent 协作的时序关系：

```typescript
type AgentInfo = {
  agentId: string
  agentName: string
  parentAgentId?: string
  processId: number        // Perfetto 中的 pid
  threadId: number          // djb2Hash(agentName) 计算
}
```

Swarm 场景下的可视化效果：

```
Process 1 (Main Agent):
  Thread 1: ├── interaction ──┤
            │ ├─ llm_request ─┤│
            │ ├─ tool: Agent  ─┤│

Process 2 (Sub Agent A):
  Thread 2: │   ├─ llm_request ─┤
            │   ├─ tool: Bash  ──┤

Process 3 (Sub Agent B):
  Thread 3: │   ├─ llm_request ─┤
            │   ├─ tool: FileWrite ─┤
```

### 3.3 可视化工作流

```bash
# 1. 启用追踪
CLAUDE_CODE_PERFETTO_TRACE=1 claude

# 或指定路径
CLAUDE_CODE_PERFETTO_TRACE=/tmp/my-trace.json claude

# 2. 可选：设置周期性写入间隔（默认仅退出时写入）
CLAUDE_CODE_PERFETTO_WRITE_INTERVAL_S=10 claude

# 3. Trace 文件位置
~/.claude/traces/trace-<session-id>.json

# 4. 打开 ui.perfetto.dev 加载文件
```

---

## 4. OpenTelemetry 基础设施

### 来源

`src/utils/telemetry/instrumentation.ts` (~826 行)

### 4.1 三信号初始化

OTel 初始化遵循标准的三信号（Metrics / Logs / Traces）模式：

```typescript
// MeterProvider — 指标
import { MeterProvider, PeriodicExportingMetricReader } from '@opentelemetry/sdk-metrics'

// LoggerProvider — 日志/事件
import { LoggerProvider, BatchLogRecordProcessor } from '@opentelemetry/sdk-logs'

// TracerProvider — 追踪
import { BasicTracerProvider, BatchSpanProcessor } from '@opentelemetry/sdk-trace-base'
```

三个 Provider 分别通过 `bootstrap/state.ts` 中的全局 getter/setter 管理：

```typescript
setMeterProvider(meterProvider)
setLoggerProvider(loggerProvider)
setTracerProvider(tracerProvider)
setEventLogger(eventLogger)

// 运行时通过 getter 获取
getMeterProvider()
getLoggerProvider()
getTracerProvider()
getEventLogger()
```

### 4.2 导出器配置

导出器通过环境变量动态选择，**动态 import** 按需加载（避免全量加载 6 个 exporter 包约 1.2MB）：

```
OTEL_METRICS_EXPORTER  → console | otlp | prometheus
OTEL_LOGS_EXPORTER     → console | otlp
OTEL_TRACES_EXPORTER   → console | otlp
```

OTLP 协议变体：

```
OTEL_EXPORTER_OTLP_PROTOCOL → grpc | http/protobuf | http/json
```

导出间隔：

| 信号 | 默认间隔 |
|------|---------|
| Metrics | 60,000 ms (1 分钟) |
| Logs | 5,000 ms (5 秒) |
| Traces | 5,000 ms (5 秒) |

### 4.3 BigQuery 自定义导出

来源：`src/utils/telemetry/bigqueryExporter.ts`

自定义 `PushMetricExporter` 实现，将聚合后的 OTel 指标推送到 Anthropic 内部 API：

```
POST /api/claude_code/metrics
```

关键特性：
- **组织级 Opt-Out** — 通过 `checkMetricsEnabled()` API 检查组织是否允许
- **缓存** — 指标启用状态缓存到本地，减少 API 调用
- **独立于标准 OTLP** — 不依赖用户配置的 OTEL 导出器

### 4.4 延迟初始化与关闭

**初始化时机** — 遥测在 trust 建立之后才初始化（`src/entrypoints/init.ts`），启动阶段的事件通过 Analytics 队列缓冲：

```
CLI 启动 → 信任检查 → initializeTelemetry() → drain 队列
```

**关闭流程** — 带超时的优雅关闭：

```typescript
class TelemetryTimeoutError extends Error {}

function telemetryTimeout(ms: number, message: string): Promise<never> {
  return new Promise((_, reject) => {
    setTimeout(
      (rej, msg) => rej(new TelemetryTimeoutError(msg)),
      ms,
    )
  })
}
```

`flushTelemetry()` 在进程退出前确保所有缓冲的 metrics/logs/traces 被导出，超时后放弃并继续退出流程。

---

## 5. 指标聚合层 (Metrics)

### 5.1 全局状态计数器

来源：`src/bootstrap/state.ts`

在全局 `State` 对象中维护了完整的会话级指标：

```typescript
type State = {
  // 成本
  totalCostUSD: number
  hasUnknownModelCost: boolean

  // API 性能
  totalAPIDuration: number
  totalAPIDurationWithoutRetries: number

  // 工具性能
  totalToolDuration: number

  // Turn 级指标（每轮重置）
  turnHookDurationMs: number
  turnToolDurationMs: number
  turnClassifierDurationMs: number
  turnToolCount: number
  turnHookCount: number
  turnClassifierCount: number

  // 代码变更
  totalLinesAdded: number
  totalLinesRemoved: number

  // 时间
  startTime: number
  lastInteractionTime: number

  // 模型使用统计
  modelUsage: { [modelName: string]: ModelUsage }
}
```

### 5.2 OTel Meter 指标

通过 `getMeter()` 获取 OTel Meter，创建 `AttributedCounter` 类型的计数器：

```typescript
export type AttributedCounter = {
  add(value: number, additionalAttributes?: Attributes): void
}
```

追踪的指标维度：

| 指标 | 来源 | 维度 |
|------|------|------|
| sessions | state.ts | subscription_type, platform |
| cost_usd | cost-tracker.ts | model, provider |
| tokens | cost-tracker.ts | model, direction (input/output/cache) |
| lines_of_code | diff.ts | operation (add/remove) |
| pr_created | state.ts | - |
| commit_created | state.ts | - |
| code_edit_decisions | state.ts | decision_type |
| active_time_ms | state.ts | - |

### 5.3 API Turn 指标消息

来源：`src/utils/messages.ts`

每轮 API 交互结束时生成 `SystemApiMetricsMessage`，包含精细的性能分解：

```typescript
type SystemApiMetricsMessage = {
  type: 'system'
  subtype: 'api_metrics'
  ttft_ms: number          // Time To First Token
  otps: number             // Output Tokens Per Second
  turn_duration_ms: number // 完整 turn 耗时
  hook_duration_ms: number // 钩子总耗时
  tool_duration_ms: number // 工具总耗时
  classifier_duration_ms: number  // 分类器总耗时
  tool_count: number
  hook_count: number
  classifier_count: number
}
```

这些指标同时通过 `logEvent` 和 `logOTelEvent` 双重上报。

---

## 6. 事件分析层 (Analytics)

### 6.1 零依赖公共 API

来源：`src/services/analytics/index.ts`

设计目标：**零依赖**，避免导入循环。整个模块只导出类型和函数，不导入任何业务模块。

```typescript
// Sink 接口
export type AnalyticsSink = {
  logEvent: (eventName: string, metadata: LogEventMetadata) => void
  logEventAsync: (eventName: string, metadata: LogEventMetadata) => Promise<void>
}

// 公共 API — 唯一的事件上报入口
export function logEvent(eventName: string, metadata: LogEventMetadata): void
export async function logEventAsync(eventName: string, metadata: LogEventMetadata): Promise<void>
```

**启动前队列缓冲** — Sink 未初始化时事件进队列，attach 后通过 `queueMicrotask` 异步 drain，既不丢失早期事件，也不阻塞启动路径：

```typescript
const eventQueue: QueuedEvent[] = []
let sink: AnalyticsSink | null = null

export function attachAnalyticsSink(newSink: AnalyticsSink): void {
  if (sink !== null) return  // 幂等
  sink = newSink

  if (eventQueue.length > 0) {
    const queuedEvents = [...eventQueue]
    eventQueue.length = 0

    queueMicrotask(() => {
      for (const event of queuedEvents) {
        if (event.async) {
          void sink!.logEventAsync(event.eventName, event.metadata)
        } else {
          sink!.logEvent(event.eventName, event.metadata)
        }
      }
    })
  }
}
```

### 6.2 Sink 路由架构

来源：`src/services/analytics/sink.ts`

```
logEvent(name, metadata)
       │
       ▼
  shouldSampleEvent(name)
       │
       ├── return 0 → 丢弃（未被采样选中）
       │
       ├── return N → 添加 sample_rate: N 到 metadata
       │   │
       │   ├──→ shouldTrackDatadog()
       │   │    ├── isSinkKilled('datadog') → 被 kill switch 关闭
       │   │    ├── isDatadogGateEnabled → Statsig gate 控制
       │   │    └── YES → stripProtoFields(metadata) → trackDatadogEvent()
       │   │
       │   └──→ logEventTo1P(name, metadataWithSampleRate)
       │        └── 完整 payload（含 _PROTO_* 字段）
       │
       └── return null → 不采样，原样传递
```

### 6.3 Datadog 通道

来源：`src/services/analytics/datadog.ts`

- **白名单模式** — 只有 `DATADOG_ALLOWED_EVENTS` 中的事件名才会发送
- **HTTP Logs Intake** — 通过 Datadog Logs API 直接推送
- **批量发送** + 周期性 flush
- **Statsig Gate 控制** — `tengu_log_datadog_events` 门控开关
- **Kill Switch** — `sinkKillswitch.ts` 提供紧急禁用

### 6.4 第一方事件管道

来源：`src/services/analytics/firstPartyEventLogger.ts` + `firstPartyEventLoggingExporter.ts`

第一方管道接收**完整 payload**（包括 `_PROTO_*` 字段），导出器将这些特殊键提升为 proto 顶层字段后存入 BigQuery 的 PII 标记列（需特权访问）。

### 6.5 采样与门控

**事件采样**：

```typescript
function shouldSampleEvent(eventName: string): number | null {
  // 从 'tengu_event_sampling_config' 动态配置读取采样率
  // 返回值:
  //   0     → 不记录（未被选中）
  //   N > 0 → 记录，附加 sample_rate = N
  //   null  → 不采样，全量记录
}
```

**Feature Gate**：

- **GrowthBook** — 当前主力，实验/AB 测试
- **Statsig（Legacy）** — 缓存式 gate 查询 `checkStatsigFeatureGate_CACHED_MAY_BE_STALE()`，使用上一个 session 的缓存值，避免初始化延迟

### 6.6 PII 防护机制

两层类型系统防护：

```typescript
// 标记 1: 通用 metadata — 强制确认不含代码/文件路径
export type AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS = never

// 标记 2: PII 标记字段 — 仅流向 1P 特权列
export type AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED = never
```

`_PROTO_*` 键的生命周期：

```
logEvent({ ..., _PROTO_user_name: "..." })
       │
       ├──→ Datadog: stripProtoFields() 剥离 _PROTO_* 键 → 安全
       │
       └──→ 1P Exporter: 提升 _PROTO_* 到 proto 顶层字段
            → 写入 BQ PII 标记列（特权访问）
            → stripProtoFields() 防止残留到 additional_metadata JSON blob
```

---

## 7. 多级日志层 (Logging)

### 7.1 Debug 日志

来源：`src/utils/debug.ts`

```typescript
export function logForDebugging(
  message: string,
  options?: { level?: 'debug' | 'info' | 'warn' | 'error' }
): void
```

特性：
- **分级过滤** — `CLAUDE_CODE_DEBUG_LOG_LEVEL` 环境变量控制
- **文件输出** — 写入 `~/.claude/logs/` 目录
- **Latest 软链接** — 最新日志文件 symlink 到 `latest`，方便 `tail -f`
- **`--debug` 标志** — CLI 参数启用详细日志

### 7.2 诊断日志 (Diagnostics)

来源：`src/utils/diagLogs.ts`

专为容器环境设计的结构化日志，通过环境管理器上报到 session-ingress：

```typescript
type DiagnosticLogEntry = {
  timestamp: string
  level: DiagnosticLogLevel    // 'debug' | 'info' | 'warn' | 'error'
  event: string                // 事件名: "started", "mcp_connected", etc.
  data: Record<string, unknown>
}

// 核心约束: 此函数绝不能传入 PII（文件路径、项目名、prompt 等）
export function logForDiagnosticsNoPII(
  level: DiagnosticLogLevel,
  event: string,
  data?: Record<string, unknown>,
): void
```

**Timing 包装器** — 自动记录操作的开始/完成/失败及耗时：

```typescript
export async function withDiagnosticsTiming<T>(
  event: string,
  fn: () => Promise<T>,
  getData?: (result: T) => Record<string, unknown>,
): Promise<T> {
  const startTime = Date.now()
  logForDiagnosticsNoPII('info', `${event}_started`)

  try {
    const result = await fn()
    logForDiagnosticsNoPII('info', `${event}_completed`, {
      duration_ms: Date.now() - startTime,
      ...getData?.(result) ?? {},
    })
    return result
  } catch (error) {
    logForDiagnosticsNoPII('error', `${event}_failed`, {
      duration_ms: Date.now() - startTime,
    })
    throw error
  }
}
```

输出位置：`CLAUDE_CODE_DIAGNOSTICS_FILE` 环境变量指定，JSONL 格式。

### 7.3 错误日志

来源：`src/utils/log.ts` + `src/utils/errorLogSink.ts`

双层存储：

```
logError(error)
   │
   ├── 内存环形缓冲 (MAX_IN_MEMORY_ERRORS = 100)
   │   └── 最近错误快速查询
   │
   └── ErrorLogSink
       ├── JSONL 文件持久化 (~/.claude/cache/)
       └── MCP 日志
```

**隐私控制** — 以下场景跳过错误上报：
- `DISABLE_ERROR_REPORTING` 环境变量
- Bedrock / Vertex / Foundry 等非直连 API
- `isEssentialTrafficOnly()` 返回 true

### 7.4 OTel 事件日志

来源：`src/utils/telemetry/events.ts`

通过 OTel `LoggerProvider` 的 `EventLogger` 发射结构化事件：

```typescript
export async function logOTelEvent(
  eventName: string,
  metadata: { [key: string]: string | undefined } = {},
): Promise<void> {
  const attributes: Attributes = {
    ...getTelemetryAttributes(),
    'event.name': eventName,
    'event.timestamp': new Date().toISOString(),
    'event.sequence': eventSequence++,  // 单调递增，保证会话内排序
  }

  // prompt.id 仅附加到事件（非指标），避免高基数
  const promptId = getPromptId()
  if (promptId) {
    attributes['prompt.id'] = promptId
  }

  eventLogger.emit({
    body: `claude_code.${eventName}`,
    attributes,
  })
}
```

**用户输入隐私** — `OTEL_LOG_USER_PROMPTS` 环境变量控制是否记录用户 prompt 原文，默认 `<REDACTED>`：

```typescript
export function redactIfDisabled(content: string): string {
  return isUserPromptLoggingEnabled() ? content : '<REDACTED>'
}
```

---

## 8. 性能剖析 (Profiling)

### 8.1 启动剖析器

来源：`src/utils/startupProfiler.ts`

两种模式：

| 模式 | 触发方式 | 覆盖率 | 输出 |
|------|---------|--------|------|
| Sampled Logging | 自动 | 100% ant / 0.5% external | Statsig 事件 |
| Detailed Profiling | `CLAUDE_CODE_PROFILE_STARTUP=1` | 100%（手动启用） | 文件报告 + 内存快照 |

追踪的启动阶段：

```typescript
const PHASE_DEFINITIONS = {
  import_time: ['cli_entry', 'main_tsx_imports_loaded'],
  init_time: ['init_function_start', 'init_function_end'],
  settings_time: ['eagerLoadSettings_start', 'eagerLoadSettings_end'],
  total_time: ['cli_entry', 'main_after_run'],
}
```

基于 Node.js Performance Hooks API (`performance.mark()` / `performance.measure()`)：

```typescript
export function profileCheckpoint(name: string): void {
  if (!SHOULD_PROFILE) return
  getPerformance().mark(name)
  if (DETAILED_PROFILING) {
    memorySnapshots.push(process.memoryUsage())
  }
}
```

### 8.2 Headless 剖析器

来源：`src/utils/headlessProfiler.ts`

专门针对非交互模式 (`claude -p "..."`) 的性能追踪：
- Per-turn 延迟
- Performance marks
- 上报到 Statsig

### 8.3 Query 剖析器

来源：`src/utils/queryProfiler.ts`

API 请求管道各阶段的耗时分解：

```
网络延迟 vs 处理阶段 vs 重试等待
```

### 8.4 慢操作检测

来源：`src/utils/slowOperations.ts`

基于 `performance.now()` 的运行时检测：

```typescript
// 包装常见慢操作
export function jsonStringify(value: unknown): string {
  const start = performance.now()
  const result = JSON.stringify(value)
  const duration = performance.now() - start
  if (duration > threshold) {
    addSlowOperation('jsonStringify', duration)
  }
  return result
}
```

检测目标：
- 大对象 JSON 序列化
- 深拷贝操作
- 可配置的阈值

---

## 9. 健康保活与异常退出

### 9.1 远程会话 Heartbeat

来源：`src/cli/transports/ccrClient.ts`, `src/bridge/bridgeMain.ts`

远程（bridge）模式下，worker 定期向服务端发送 HTTP heartbeat 延长 lease：

```
Worker → POST .../heartbeat → Bridge Server
         interval + jitter
```

关联事件：`tengu_bridge_heartbeat_*`

### 9.2 本地会话活动追踪

来源：`src/utils/sessionActivity.ts`

基于引用计数的活动检测机制：

```
活动开始 → refCount++
活动结束 → refCount--
定时器检查 → refCount > 0 ? 写入 session_keepalive_heartbeat : skip
```

心跳写入 `CLAUDE_CODE_DIAGNOSTICS_FILE`。

### 9.3 异常退出处理

来源：`src/utils/gracefulShutdown.ts`

```typescript
// 全局异常捕获
process.on('uncaughtException', handler)
process.on('unhandledRejection', handler)
```

处理流程：

```
异常捕获
   │
   ├── logForDiagnosticsNoPII('error', ...)  → 诊断日志
   ├── logEvent('tengu_uncaught_exception')   → Analytics
   ├── logEvent('tengu_unhandled_rejection')  → Analytics
   │
   ├── 孤儿进程检测
   │   └── TTY stdin/stdout 状态检查
   │       └── logForDiagnosticsNoPII('warn', 'orphan_process_detected')
   │
   └── 优雅关闭序列
       ├── endInteractionSpan()         → 关闭未结束的 span
       ├── flushTelemetry()             → 刷新遥测缓冲
       ├── shutdownDatadog()            → 关闭 Datadog 连接
       ├── shutdown1PEventLogging()     → 关闭 1P 事件管道
       ├── runCleanupFunctions()        → 执行注册的清理函数
       └── profileReport()             → 输出启动剖析报告
```

---

## 10. 隐私与开关控制

Claude Code 的监控系统有极细粒度的隐私控制，形成三级递进：

```
┌─────────────────────────────────────────────────────────┐
│ Level 1: DISABLE_TELEMETRY=1                            │
│   → 完全禁用所有遥测（OTel + Analytics + Diagnostics）   │
├─────────────────────────────────────────────────────────┤
│ Level 2: isEssentialTrafficOnly()                        │
│   → 仅保留必要的错误上报，禁用产品分析类事件               │
├─────────────────────────────────────────────────────────┤
│ Level 3: Privacy Level (privacyLevel.ts)                 │
│   → 细粒度控制各通道的启用/禁用                           │
└─────────────────────────────────────────────────────────┘
```

关键环境变量：

| 环境变量 | 作用 |
|---------|------|
| `DISABLE_TELEMETRY` | 完全禁用遥测 |
| `CLAUDE_CODE_ENABLE_TELEMETRY` | 显式启用遥测 |
| `DISABLE_ERROR_REPORTING` | 禁用错误上报 |
| `OTEL_LOG_USER_PROMPTS` | 是否记录用户 prompt 原文 |
| `CLAUDE_CODE_ENHANCED_TELEMETRY_BETA` | 增强追踪开关 |
| `CLAUDE_CODE_PERFETTO_TRACE` | Perfetto 本地追踪 |
| `CLAUDE_CODE_DIAGNOSTICS_FILE` | 诊断日志文件路径 |
| `CLAUDE_CODE_DEBUG_LOG_LEVEL` | Debug 日志级别 |
| `CLAUDE_CODE_PROFILE_STARTUP` | 启动剖析 |
| `OTEL_METRICS_EXPORTER` | Metrics 导出器选择 |
| `OTEL_LOGS_EXPORTER` | Logs 导出器选择 |
| `OTEL_TRACES_EXPORTER` | Traces 导出器选择 |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | OTLP 协议变体 |

**API Provider 差异化** — Bedrock / Vertex / Foundry 等非直连 API 场景自动降低遥测级别：

```typescript
// log.ts 中的条件检查
if (isEssentialTrafficOnly()) {
  // 跳过非必要的错误上报
}
```

**组织级 Opt-Out** — `metricsOptOut.ts` 通过 API 检查组织是否允许指标上报，结果缓存到本地 `metrics_enabled` 标志。

---

## 11. 全链路数据流

一次完整用户交互的监控数据流：

```
用户输入 "fix the bug in auth.ts"
│
├─── [追踪层] startInteractionSpan()
│    ├── OTel Span: claude_code.interaction (attributes: prompt_length, sequence)
│    └── Perfetto: Begin 'interaction' event
│
├─── [事件层] logEvent('tengu_interaction_start', {...})
│    ├── → Datadog (if allowlisted + gate enabled)
│    └── → 1P Event Pipeline → BigQuery
│
├─── LLM API 调用
│    │
│    ├── [追踪层] startLLMRequestSpan(model, newContext, messages)
│    │   ├── OTel Span: claude_code.llm_request (child of interaction)
│    │   └── Perfetto: Begin 'api_request' event
│    │
│    ├── [剖析层] queryProfiler 开始计时
│    │
│    ├── [指标层] Streaming 中计算 TTFT
│    │
│    ├── [追踪层] endLLMRequestSpan(usage, stopReason, ...)
│    │   ├── OTel Span attributes: model, usage.*, duration_ms
│    │   └── Perfetto: End event (args: TTFT, TTLT, cache_stats)
│    │
│    ├── [指标层] cost counter.add(cost, {model})
│    ├── [指标层] token counter.add(tokens, {model, direction})
│    │
│    └── [事件层] logEvent('tengu_query_success', {ttft_ms, otps, ...})
│         logOTelEvent('query_completed', {...})
│
├─── 工具调用: FileRead("auth.ts")
│    │
│    ├── [追踪层] startToolSpan('FileRead')
│    │   └── OTel Span: claude_code.tool (child of interaction)
│    │
│    ├── [权限检查]
│    │   └── 如需用户确认:
│    │       ├── startBlockedOnUserSpan()
│    │       └── endBlockedOnUserSpan(duration)
│    │
│    ├── [追踪层] startToolExecutionSpan()
│    │   └── 实际文件读取
│    │
│    ├── [追踪层] endToolSpan(result)
│    │   └── Perfetto: End tool event
│    │
│    └── [事件层] logEvent('tengu_tool_use', {tool: 'FileRead', duration_ms})
│
├─── 工具调用: FileWrite("auth.ts", newContent)
│    └── (同上流程)
│       └── [指标层] linesAdded counter.add(N)
│                   linesRemoved counter.add(M)
│
├─── Turn 结束
│    │
│    ├── [指标层] createApiMetricsMessage({
│    │     ttft_ms, otps, turn_duration_ms,
│    │     hook_duration_ms, tool_duration_ms, ...
│    │   })
│    │
│    └── [追踪层] endInteractionSpan()
│        ├── OTel: span.end() with interaction.duration_ms
│        └── Perfetto: End 'interaction' event
│
└─── [导出]
     ├── PeriodicExportingMetricReader → BigQuery / OTLP / Prometheus
     ├── BatchLogRecordProcessor → OTLP Console
     ├── BatchSpanProcessor → OTLP Console
     ├── Datadog HTTP flush
     └── 1P Event batch export
```

---

## 12. 设计模式总结

### 模式 1: 零开销降级

未启用追踪时，Span 操作退化为 dummy span / no-op，不影响业务性能：

```typescript
if (!isAnyTracingEnabled()) {
  return trace.getActiveSpan() || getTracer().startSpan('dummy')
}
```

### 模式 2: 双格式并行追踪

OTel（标准化、可导出到任意后端）+ Perfetto（本地深度分析、Chrome 可视化）在每个 span 入口同时启动，互不干扰：

```typescript
export function startInteractionSpan(userPrompt: string): Span {
  // Perfetto — 独立于 OTel 状态
  const perfettoSpanId = isPerfettoTracingEnabled()
    ? startInteractionPerfettoSpan(userPrompt)
    : undefined

  // OTel — 标准追踪
  if (!isAnyTracingEnabled()) { /* ... */ }
  const span = tracer.startSpan('claude_code.interaction', { attributes })
  // ...
}
```

### 模式 3: 类型系统防 PII 泄漏

通过 `never` 类型标记强制开发者在使用字符串时显式声明安全性，编译期即可捕获潜在的 PII 泄漏：

```typescript
export type AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS = never
```

### 模式 4: 启动前队列 + 延迟 Sink

Analytics 和 Telemetry 都采用了"先队列后 drain"的模式：

```
logEvent() → queue (sink 未就绪)
     ...
attachAnalyticsSink() → queueMicrotask drain
```

确保启动阶段的事件不丢失，又不阻塞启动路径。

### 模式 5: 多级隐私分层

从完全禁用到细粒度控制，逐级递进：

```
DISABLE_TELEMETRY → isEssentialTrafficOnly() → privacyLevel → per-field _PROTO_*
```

### 模式 6: 缓存式 Gate 查询

Feature gate 使用上一个 session 的缓存值，避免启动时的网络延迟：

```typescript
checkStatsigFeatureGate_CACHED_MAY_BE_STALE(gateName)
```

方法名中的 `_CACHED_MAY_BE_STALE` 后缀清晰地告知调用者：返回值可能是过期的。

### 模式 7: Kill Switch 紧急开关

每个上报通道都有独立的紧急禁用能力：

```typescript
if (isSinkKilled('datadog')) return false
```

当某个后端出问题时，可以远程关闭而不影响其他通道。

### 模式 8: Swarm 感知的进程映射

Perfetto 追踪内置多 Agent 的 process/thread 映射，使用 `djb2Hash(agentName)` 为每个 Agent 生成稳定的 thread ID，在 Perfetto UI 中自然呈现为并行的 "线程"。
