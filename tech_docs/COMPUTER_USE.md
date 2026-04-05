---
title: Computer Use
aliases: [Computer Use, 屏幕交互, Chicago]
series: Claude Code 架构解析
category: 特色功能
order: 21
tags:
  - claude-code
  - computer-use
  - macos
  - swift
  - rust
  - screen-interaction
date: 2026-04-04
---

# Computer Use 详细技术文档

## 1. 概述

Computer Use（代号 **Chicago**）是 Claude Code 的屏幕交互能力，允许 Claude 通过截图观察用户屏幕并通过模拟鼠标/键盘操作来控制 macOS 应用程序。整个系统以 **MCP 服务器**形式暴露（非一等工具），工具名为 `mcp__computer-use__*`，由 `@ant/computer-use-mcp` 包提供核心逻辑，CLI 端负责**宿主适配、门控、MCP 接线和 UI**。

### 核心依赖

| 包 | 语言 | 职责 |
|----|------|------|
| `@ant/computer-use-mcp` | TypeScript | 工具 schema、会话绑定、坐标变换、权限流程 |
| `@ant/computer-use-swift` | Swift | SCContentFilter 截图、NSWorkspace 应用管理、TCC 检查 |
| `@ant/computer-use-input` | Rust (enigo) | 鼠标移动/点击/滚动/拖拽、键盘输入 |

### 平台限制

- **仅 macOS** — `createCliExecutor` 在非 darwin 平台直接抛出异常
- 需要 **Accessibility** 和 **Screen Recording** TCC 权限

---

## 2. 架构总览

```
┌──────────────────────────────────────────────────────────────────┐
│                        启用门控                                    │
│                                                                    │
│  feature('CHICAGO_MCP')  ── 编译时 DCE                            │
│  getChicagoEnabled()     ── 运行时 GrowthBook + 订阅检查          │
│  getPlatform() === 'macos' ── 平台检查                            │
│  非 headless 模式         ── 交互式检查                            │
└─────────────────────┬────────────────────────────────────────────┘
                      │ 全部通过
                      ▼
┌──────────────────────────────────────────────────────────────────┐
│              setup.ts — setupComputerUseMCP()                     │
│                                                                    │
│  buildComputerUseTools(CLI_CU_CAPABILITIES, coordinateMode)       │
│  → 生成工具列表 → 映射为 mcp__computer-use__<name>               │
│  → 返回 { mcpConfig, allowedTools }                              │
│                                                                    │
│  mcpConfig: { 'computer-use': { type: 'stdio', ... } }          │
│  allowedTools: 绕过权限提示 (package 自己的 request_access 处理)   │
└─────────────────────┬────────────────────────────────────────────┘
                      │
                      ▼
┌──────────────────────────────────────────────────────────────────┐
│           mcp/client.ts — 进程内 MCP 连接                         │
│                                                                    │
│  检测 server name = 'computer-use'                                │
│  → 不启动子进程，改用进程内 MCP stub server                       │
│  → 工具对象 spread getComputerUseMCPToolOverrides(tool.name)     │
│    → .call() 覆写为 wrapper.tsx 的 dispatch                      │
│    → 渲染覆写为 toolRendering.tsx                                 │
└─────────────────────┬────────────────────────────────────────────┘
                      │
                      ▼
┌──────────────────────────────────────────────────────────────────┐
│         wrapper.tsx — .call() 覆写 (会话绑定层)                   │
│                                                                    │
│  getOrBind()                                                       │
│    └─ bindSessionContext(hostAdapter, coordinateMode, ctx)         │
│       → dispatch(toolName, args)                                   │
│                                                                    │
│  结果映射:                                                         │
│    MCP image → { type: 'image', source: { type: 'base64', ... }} │
│    MCP text  → { type: 'text', text: '...' }                     │
│                                                                    │
│  权限对话框:                                                       │
│    runPermissionDialog() → ComputerUseApproval React 组件         │
└─────────────────────┬────────────────────────────────────────────┘
                      │
                      ▼
┌──────────────────────────────────────────────────────────────────┐
│     hostAdapter.ts — ComputerUseHostAdapter 单例                  │
│                                                                    │
│  executor: createCliExecutor(...)  ← executor.ts                  │
│  ensureOsPermissions: TCC 检查                                     │
│  isDisabled: () => !getChicagoEnabled()                           │
│  getSubGates: 动态子门控                                           │
│  cropRawPatch: () => null (像素验证跳过)                          │
└─────────────────────┬────────────────────────────────────────────┘
                      │
                      ▼
┌──────────────────────────────────────────────────────────────────┐
│            executor.ts — ComputerExecutor 实现                     │
│                                                                    │
│  ┌───────────────┐  ┌──────────────────┐  ┌─────────────────┐   │
│  │ @ant/computer- │  │ @ant/computer-   │  │ pbcopy/pbpaste │   │
│  │ use-swift      │  │ use-input (Rust) │  │ (系统剪贴板)   │   │
│  │                │  │                  │  │                 │   │
│  │ · screenshot   │  │ · moveMouse      │  │ · 读/写文本    │   │
│  │ · display info │  │ · mouseButton    │  │ · 粘贴验证     │   │
│  │ · app管理      │  │ · mouseScroll    │  └─────────────────┘   │
│  │ · TCC检查      │  │ · key/keys       │                        │
│  │ · prepareDisp. │  │ · typeText       │                        │
│  └───────────────┘  │ · getFrontmost   │                        │
│                      │ · mouseLocation  │                        │
│                      └──────────────────┘                        │
└──────────────────────────────────────────────────────────────────┘
```

---

## 3. 启用门控 (Feature Gates)

Computer Use 由多层门控保护，必须全部通过才能激活：

### 3.1 编译时门控

```typescript
feature('CHICAGO_MCP')  // bun:bundle 编译时特性
```

所有 CU 相关的 `import` 和代码路径都被 `feature('CHICAGO_MCP')` 包裹。编译时 DCE (Dead Code Elimination) 确保未启用该特性的构建不包含任何 CU 代码。

### 3.2 运行时门控 — GrowthBook

```typescript
// gates.ts
function readConfig(): ChicagoConfig {
  return {
    ...DEFAULTS,
    ...getDynamicConfig_CACHED_MAY_BE_STALE<Partial<ChicagoConfig>>(
      'tengu_malort_pedway',  // GrowthBook key
      DEFAULTS,
    ),
  }
}
```

**默认配置**（全部关闭/保守值）：

| 字段 | 默认值 | 说明 |
|------|--------|------|
| `enabled` | `false` | 总开关 |
| `pixelValidation` | `false` | 点击前像素校验 |
| `clipboardPasteMultiline` | `true` | 多行粘贴走剪贴板 |
| `mouseAnimation` | `true` | 拖拽动画 (ease-out-cubic) |
| `hideBeforeAction` | `true` | 操作前隐藏其他窗口 |
| `autoTargetDisplay` | `true` | 自动选择显示器 |
| `clipboardGuard` | `true` | 剪贴板保存/恢复 |
| `coordinateMode` | `'pixels'` | 坐标模式 |

### 3.3 订阅检查

```typescript
function hasRequiredSubscription(): boolean {
  if (process.env.USER_TYPE === 'ant') return true  // Anthropic 内部绕过
  const tier = getSubscriptionType()
  return tier === 'max' || tier === 'pro'
}
```

仅 **Max** 或 **Pro** 订阅用户可用。Anthropic 内部员工 (`USER_TYPE === 'ant'`) 绕过订阅检查，但 monorepo 开发环境默认禁用（避免意外触发）。

### 3.4 平台与会话检查

```typescript
// main.tsx
if (feature('CHICAGO_MCP') && getPlatform() === 'macos' && !isNonInteractive) {
  setupComputerUseMCP()
}
```

- 仅 macOS
- 仅交互式会话（排除 SDK/headless）
- MCP server 名 `computer-use` 被**保留**（用户不能添加同名 MCP）

### 3.5 坐标模式冻结

```typescript
let frozenCoordinateMode: CoordinateMode | undefined
export function getChicagoCoordinateMode(): CoordinateMode {
  frozenCoordinateMode ??= readConfig().coordinateMode
  return frozenCoordinateMode
}
```

坐标模式在首次读取后**冻结**——防止 GrowthBook 在会话中途切换模式导致工具描述（告诉模型用 pixels）与执行器（按 normalized 变换坐标）不一致。

---

## 4. 截图系统

### 4.1 截图捕获

```typescript
// executor.ts
async screenshot(opts: {
  allowedBundleIds: string[]
  displayId?: number
}): Promise<ScreenshotResult> {
  const d = cu.display.getSize(opts.displayId)
  const [targetW, targetH] = computeTargetDims(d.width, d.height, d.scaleFactor)
  return drainRunLoop(() =>
    cu.screenshot.captureExcluding(
      withoutTerminal(opts.allowedBundleIds),
      SCREENSHOT_JPEG_QUALITY,  // 0.75
      targetW, targetH,
      opts.displayId,
    )
  )
}
```

**关键流程**：

1. **获取显示器尺寸**：`cu.display.getSize(displayId)` → 逻辑宽高 + scaleFactor
2. **计算目标尺寸**：`computeTargetDims()` — 逻辑 × scaleFactor = 物理 → `targetImageSize(physW, physH, API_RESIZE_PARAMS)`
3. **捕获**：`cu.screenshot.captureExcluding()` — SCContentFilter 捕获（允许列表语义）
4. **终端排除**：`withoutTerminal()` 过滤掉终端 bundleId，确保终端不出现在截图中
5. **Run Loop 泵送**：`drainRunLoop()` 确保 Swift `@MainActor` 工作完成

### 4.2 尺寸计算与坐标一致性

```typescript
function computeTargetDims(logicalW, logicalH, scaleFactor): [number, number] {
  const physW = Math.round(logicalW * scaleFactor)
  const physH = Math.round(logicalH * scaleFactor)
  return targetImageSize(physW, physH, API_RESIZE_PARAMS)
}
```

截图被预缩放到 `targetImageSize` 的输出尺寸，使得 API 转码器的 early-return 触发（不做服务端 resize），从而 `scaleCoord` 保持一致。参见 `@ant/computer-use-mcp/COORDINATES.md`。

### 4.3 区域截图 (Zoom)

```typescript
async zoom(regionLogical, allowedBundleIds, displayId): Promise<...> {
  const [outW, outH] = computeTargetDims(regionLogical.w, regionLogical.h, d.scaleFactor)
  return drainRunLoop(() =>
    cu.screenshot.captureRegion(
      withoutTerminal(allowedBundleIds),
      regionLogical.x, regionLogical.y,
      regionLogical.w, regionLogical.h,
      outW, outH,
      SCREENSHOT_JPEG_QUALITY, displayId
    )
  )
}
```

### 4.4 截图到 LLM

```typescript
// wrapper.tsx — 结果映射
const data = result.content.map(item =>
  item.type === 'image'
    ? {
        type: 'image',
        source: {
          type: 'base64',
          media_type: item.mimeType ?? 'image/jpeg',
          data: item.data
        }
      }
    : { type: 'text', text: item.type === 'text' ? item.text : '' }
)
```

CU 截图以 **pre-sized JPEG** (质量 0.75) 返回，无需通用 MCP 路径的额外 resize。MCP image 块直接映射为 Anthropic API 的 base64-source image 块。

---

## 5. 鼠标操作

### 5.1 移动

```typescript
async moveMouse(x, y): Promise<void> {
  await moveAndSettle(requireComputerUseInput(), x, y)
}

// moveAndSettle: 即时移动 + 50ms 等待 HID→AppKit→NSEvent 往返
async function moveAndSettle(input, x, y): Promise<void> {
  await input.moveMouse(x, y, false)
  await sleep(MOVE_SETTLE_MS)  // 50ms
}
```

### 5.2 点击

```typescript
async click(x, y, button, count, modifiers?): Promise<void> {
  await moveAndSettle(input, x, y)
  if (modifiers?.length) {
    await drainRunLoop(() =>
      withModifiers(input, modifiers, () =>
        input.mouseButton(button, 'click', count)
      )
    )
  } else {
    await input.mouseButton(button, 'click', count)
  }
}
```

- 支持 `left` / `right` / `middle` 按钮
- 支持单击、双击、三击（`count: 1 | 2 | 3`）
- AppKit 通过时间+位置自动计算 `clickCount`
- 修饰键通过 `withModifiers()` bracket 模式确保 press/release 配对

### 5.3 拖拽

```typescript
async drag(from, to): Promise<void> {
  if (from) await moveAndSettle(input, from.x, from.y)
  await input.mouseButton('left', 'press')
  await sleep(MOVE_SETTLE_MS)  // 等待 HID 注册
  try {
    await animatedMove(input, to.x, to.y, getMouseAnimationEnabled())
  } finally {
    await input.mouseButton('left', 'release')  // 无论如何释放
  }
}
```

**拖拽动画** (`animatedMove`)：
- Ease-out-cubic 曲线：`1 - (1 - t)³`
- 60fps、2000 px/s、最长 0.5s
- 仅拖拽使用（click/scroll 用 instant move）
- 可通过 `mouseAnimation` 子门控禁用

### 5.4 滚动

```typescript
async scroll(x, y, dx, dy): Promise<void> {
  await moveAndSettle(input, x, y)
  if (dy !== 0) await input.mouseScroll(dy, 'vertical')   // 先垂直
  if (dx !== 0) await input.mouseScroll(dx, 'horizontal') // 后水平
}
```

---

## 6. 键盘操作

### 6.1 按键序列

```typescript
async key(keySequence: string, repeat?: number): Promise<void> {
  const parts = keySequence.split('+').filter(p => p.length > 0)
  const isEsc = isBareEscape(parts)
  await drainRunLoop(async () => {
    for (let i = 0; i < n; i++) {
      if (i > 0) await sleep(8)  // 125Hz USB 轮询节奏
      if (isEsc) notifyExpectedEscape()  // 通知 CGEventTap
      await input.keys(parts)
    }
  })
}
```

- xdotool 风格：`"ctrl+shift+a"` → `['ctrl', 'shift', 'a']`
- 8ms 间隔 = 125Hz USB 轮询频率
- bare Escape 通知 CGEventTap，避免误触发中止回调

### 6.2 长按

```typescript
async holdKey(keyNames: string[], durationMs: number): Promise<void> {
  const pressed: string[] = []
  let orphaned = false
  try {
    await drainRunLoop(async () => {
      for (const k of keyNames) {
        if (orphaned) return
        if (isBareEscape([k])) notifyExpectedEscape()
        await input.key(k, 'press')
        pressed.push(k)
      }
    })
    await sleep(durationMs)  // 在 drainRunLoop 外等待
  } finally {
    orphaned = true
    await drainRunLoop(() => releasePressed(input, pressed))
  }
}
```

- `orphaned` 标志防止超时后孤儿 lambda 继续按键
- `durationMs` 在 drainRunLoop 外等待，不受 30s 限制

### 6.3 文本输入

```typescript
async type(text: string, opts: { viaClipboard: boolean }): Promise<void> {
  if (opts.viaClipboard) {
    await drainRunLoop(() => typeViaClipboard(input, text))
    return
  }
  await input.typeText(text)
}
```

**剪贴板粘贴流程** (`typeViaClipboard`)：

1. `pbpaste` 保存用户剪贴板
2. `pbcopy` 写入目标文本
3. **Read-back 校验** — 剪贴板写入可能静默失败，校验防止粘贴垃圾内容
4. `Cmd+V` 粘贴
5. 100ms 等待 — 粘贴效果与剪贴板恢复的竞争阈值
6. `finally` 恢复用户剪贴板

---

## 7. 显示器管理

### 7.1 多显示器支持

```typescript
async listDisplays(): Promise<DisplayGeometry[]> {
  return cu.display.listAll()
}

async getDisplaySize(displayId?): Promise<DisplayGeometry> {
  return cu.display.getSize(displayId)
}

async findWindowDisplays(bundleIds): Promise<Array<{
  bundleId: string; displayIds: number[]
}>> {
  return cu.apps.findWindowDisplays(bundleIds)
}
```

### 7.2 显示器选择策略

状态通过 `AppState.computerUseMcpState` 管理：

| 字段 | 说明 |
|------|------|
| `selectedDisplayId` | 当前活跃显示器 |
| `displayPinnedByModel` | 模型通过 `switch_display(name)` 锁定的显示器 |
| `displayResolvedForApps` | 基于已授权应用自动解析的显示器 |
| `lastScreenshotDims` | 最近截图的尺寸信息 (用于坐标缩放) |

- `switch_display(name)` — 锁定到指定显示器
- `switch_display("auto")` — 解锁，下次截图自动解析
- 当锁定的显示器被拔出，自动回退到主显示器并清除锁定

---

## 8. 应用管理

### 8.1 应用发现

```typescript
async listInstalledApps(): Promise<InstalledApp[]> {
  return drainRunLoop(() => cu.apps.listInstalled())
}

async listRunningApps(): Promise<RunningApp[]> {
  return cu.apps.listRunning()
}

async getFrontmostApp(): Promise<FrontmostApp | null> {
  const info = requireComputerUseInput().getFrontmostAppInfo()
  return info?.bundleId ? { bundleId: info.bundleId, displayName: info.appName } : null
}
```

### 8.2 操作前隐藏 (prepareForAction)

```typescript
async prepareForAction(allowlistBundleIds, displayId?): Promise<string[]> {
  if (!getHideBeforeActionEnabled()) return []
  return drainRunLoop(async () => {
    const result = await cu.apps.prepareDisplay(
      allowlistBundleIds, surrogateHost, displayId
    )
    return result.hidden
  })
}
```

操作前隐藏非目标窗口，减少视觉干扰。终端作为**代理宿主** (`surrogateHost`) 被豁免隐藏和截图排除。

### 8.3 Turn 结束清理

```typescript
// cleanup.ts
export async function cleanupComputerUseAfterTurn(ctx): Promise<void> {
  // 1. 自动取消隐藏 (unhide) 被隐藏的应用
  const hidden = appState.computerUseMcpState?.hiddenDuringTurn
  if (hidden?.size > 0) {
    await unhideComputerUseApps([...hidden])  // 5s 超时
  }

  // 2. 注销 Esc 热键
  unregisterEscHotkey()

  // 3. 释放文件锁
  if (await releaseComputerUseLock()) {
    ctx.sendOSNotification?.({
      message: 'Claude is done using your computer',
      notificationType: 'computer_use_exit',
    })
  }
}
```

清理在三个位置调用：
- 自然 turn 结束 (`stopHooks.ts`)
- 流式传输中止 (`query.ts` — aborted_streaming)
- 工具执行中止 (`query.ts` — aborted_tools)

---

## 9. 会话锁

### 9.1 锁机制

同一台机器上只允许**一个** Claude 会话使用 Computer Use。锁是基于文件的，位于 `~/.claude/computer-use.lock`。

```typescript
type ComputerUseLock = {
  readonly sessionId: string
  readonly pid: number
  readonly acquiredAt: number
}
```

### 9.2 锁操作

| 操作 | 函数 | 说明 |
|------|------|------|
| 检查 | `checkComputerUseLock()` | 不获取；`request_access` 使用 |
| 获取 | `tryAcquireComputerUseLock()` | O_EXCL 原子创建 |
| 释放 | `releaseComputerUseLock()` | turn 结束时调用 |
| 本地检查 | `isLockHeldLocally()` | 零系统调用；决定是否需要清理 |

### 9.3 过期锁恢复

```typescript
// checkComputerUseLock / tryAcquireComputerUseLock
if (isProcessRunning(existing.pid)) {
  return { kind: 'blocked', by: existing.sessionId }
}
// 进程已死 → 清理过期锁
await unlink(getLockPath())
```

使用 `process.kill(pid, 0)` 检测持锁进程是否存活。死进程的锁自动清理。竞争恢复通过 O_EXCL 保证原子性。

### 9.4 关闭保障

```typescript
registerCleanup(async () => {
  await releaseComputerUseLock()
})
```

即使 turn 结束清理未执行（如用户 `/exit`），shutdown handler 也会释放锁。

---

## 10. 权限系统

### 10.1 macOS TCC 权限

```typescript
// hostAdapter.ts
ensureOsPermissions: async () => {
  const accessibility = cu.tcc.checkAccessibility()
  const screenRecording = cu.tcc.checkScreenRecording()
  return accessibility && screenRecording
    ? { granted: true }
    : { granted: false, accessibility, screenRecording }
}
```

- **Accessibility** — 鼠标/键盘控制所需
- **Screen Recording** — 截图所需

### 10.2 应用授权

用户通过 `ComputerUseApproval` React 组件选择允许 Claude 操作的应用：

```typescript
// wrapper.tsx
onPermissionRequest: (req, _dialogSignal) => runPermissionDialog(req)

async function runPermissionDialog(req: CuPermissionRequest): Promise<CuPermissionResponse> {
  return new Promise((resolve, reject) => {
    setToolJSX({
      jsx: React.createElement(ComputerUseApproval, {
        request: req,
        onDone: resolve
      }),
      shouldHidePromptInput: true
    })
  })
}
```

**授权粒度**：

| 权限 | 说明 |
|------|------|
| 应用列表 | 允许操作的应用 bundleId |
| `clipboardRead` | 允许读取剪贴板 |
| `clipboardWrite` | 允许写入剪贴板 |
| `systemKeyCombos` | 允许系统快捷键 |

授权状态持久化在 `AppState.computerUseMcpState.allowedApps` 和 `grantFlags` 中。

### 10.3 应用列表安全

```typescript
// appNames.ts
export function filterAppsForDescription(apps: InstalledApp[]): InstalledApp[] {
  // 过滤噪声和潜在注入风险的应用名称
}
```

### 10.4 工具名称与 API 提示

```
mcp__computer-use__screenshot
mcp__computer-use__click
mcp__computer-use__type
mcp__computer-use__key
...
```

`mcp__computer-use__*` 工具名**不是偶然选择**——API 后端检测到这些名称后会在 system prompt 中注入 CU 可用性提示 (`COMPUTER_USE_MCP_AVAILABILITY_HINT`)。使用其他名称则不会触发该提示。Cowork 桌面版使用相同的命名。

---

## 11. Escape 热键中止

### 11.1 注册

```typescript
// escHotkey.ts
export function registerEscHotkey(onAbort: () => void): boolean {
  // CGEventTap 全局监听 Escape
  // 消费事件 (防止 prompt injection 通过 Escape 关闭对话框)
  // 返回是否注册成功 (需要 Accessibility 权限)
}
```

### 11.2 避免误中止

```typescript
function isBareEscape(parts: readonly string[]): boolean {
  if (parts.length !== 1) return false
  const lower = parts[0]!.toLowerCase()
  return lower === 'escape' || lower === 'esc'
}

// 模型发起的 Escape 按键前通知 tap
if (isEsc) notifyExpectedEscape()
```

当模型自己合成 Escape 按键时，预先 `notifyExpectedEscape()` 告知 CGEventTap 不要触发中止。仅 bare Escape 触发中止，`ctrl+escape` 等组合通过。

### 11.3 通知

获取锁时发送系统通知：

```
"Claude is using your computer · press Esc to stop"
// 或 (Accessibility 不可用时)
"Claude is using your computer · press Ctrl+C to stop"
```

释放锁时：
```
"Claude is done using your computer"
```

---

## 12. Run Loop 管理

### 12.1 问题背景

Swift 的 `@MainActor` 工作（截图、应用管理、窗口操作）需要 `DispatchQueue.main` 处理。Node.js 不运行 CFRunLoop，导致这些操作挂起。

### 12.2 解决方案

```typescript
// drainRunLoop.ts
export async function drainRunLoop<T>(fn: () => Promise<T>): Promise<T> {
  retainPump()
  try {
    return await fn()
  } finally {
    releasePump()
  }
}
```

- 1ms CFRunLoop pump 循环
- 30s 超时上限
- retain/release 引用计数（Esc 热键也持有 retain）
- Electron (Cowork) 不需要这个，因为它持续 drain CFRunLoop

---

## 13. 终端检测

```typescript
// common.ts
const TERMINAL_BUNDLE_ID_FALLBACK: Record<string, string> = {
  'iTerm.app': 'com.googlecode.iterm2',
  Apple_Terminal: 'com.apple.Terminal',
  ghostty: 'com.mitchellh.ghostty',
  kitty: 'net.kovidgoyal.kitty',
  WarpTerminal: 'dev.warp.Warp-Stable',
  vscode: 'com.microsoft.VSCode',
}

export function getTerminalBundleId(): string | null {
  // 优先使用 __CFBundleIdentifier (LaunchServices 注入)
  const cfBundleId = process.env.__CFBundleIdentifier
  if (cfBundleId) return cfBundleId
  // 回退到已知映射表
  return TERMINAL_BUNDLE_ID_FALLBACK[env.terminal ?? ''] ?? null
}
```

终端 bundleId 用于三个目的：
1. **隐藏豁免** — `prepareDisplay` 不隐藏终端
2. **激活跳过** — z-order walk 中跳过终端
3. **截图排除** — 终端不出现在截图中

**Sentinel** `CLI_HOST_BUNDLE_ID = 'com.anthropic.claude-code.cli-no-window'` 用于 package 的 frontmost gate，永不匹配真实应用。

---

## 14. 关键文件索引

| 文件 | 角色 |
|------|------|
| `src/utils/computerUse/gates.ts` | GrowthBook 门控 + 订阅检查 + 子门控 |
| `src/utils/computerUse/setup.ts` | MCP 配置构建 + allowedTools |
| `src/utils/computerUse/executor.ts` | ComputerExecutor 实现 (截图/鼠标/键盘) |
| `src/utils/computerUse/hostAdapter.ts` | HostAdapter 单例 (executor + TCC + 门控) |
| `src/utils/computerUse/wrapper.tsx` | .call() 覆写 + 会话绑定 + 权限对话框 |
| `src/utils/computerUse/common.ts` | 常量 + 终端检测 + 能力定义 |
| `src/utils/computerUse/computerUseLock.ts` | 文件锁 (O_EXCL 原子获取) |
| `src/utils/computerUse/cleanup.ts` | Turn 结束清理 (unhide + 锁释放) |
| `src/utils/computerUse/escHotkey.ts` | CGEventTap Escape 中止 |
| `src/utils/computerUse/drainRunLoop.ts` | CFRunLoop pump (30s 上限) |
| `src/utils/computerUse/swiftLoader.ts` | `@ant/computer-use-swift` 动态加载 |
| `src/utils/computerUse/inputLoader.ts` | `@ant/computer-use-input` 懒加载 |
| `src/utils/computerUse/toolRendering.tsx` | Ink UI 渲染覆写 |
| `src/utils/computerUse/appNames.ts` | 应用列表过滤 |
| `src/utils/computerUse/mcpServer.ts` | MCP server 工厂 + ListTools |
| `src/components/permissions/ComputerUseApproval/` | 权限审批 React 组件 |
| `src/services/mcp/client.ts` | 进程内 MCP 连接 + 工具覆写 |

---

## 15. 与 Cowork 桌面版的差异

| 特性 | Cowork (Electron) | CLI |
|------|-------------------|-----|
| 窗口 click-through | `BrowserWindow.setIgnoreMouseEvents(true)` | 无窗口 — 无需 |
| 宿主检测 | 使用自身 BrowserWindow | 终端作为代理宿主 |
| 剪贴板 | `Electron clipboard` | `pbcopy` / `pbpaste` |
| CFRunLoop | Electron 持续 drain | 手动 `drainRunLoop()` pump |
| 像素验证 | `nativeImage` (同步 crop) | 跳过 (`cropRawPatch: () => null`) |
| 前台检测 | 真实 frontmost gate | Sentinel bundleId (永不匹配) |

---

## 16. 关键设计决策

### 16.1 MCP 架构而非一等工具

**决策：** Computer Use 作为 MCP 服务器（`computer-use`）而非 `src/tools/` 下的一等工具。

**原因：** API 后端检测 `mcp__computer-use__*` 工具名注入 CU 可用性提示。使用一等工具的不同命名无法触发该行为。MCP 架构还与 Cowork 保持一致。

### 16.2 进程内 MCP

**决策：** MCP config 声明为 stdio，但 `client.ts` 拦截并使用进程内 server。

**原因：** 避免启动子进程的开销和复杂性。Native 模块（Swift/Rust）直接在主进程加载。

### 16.3 坐标冻结

**决策：** `coordinateMode` 在首次读取后冻结。

**原因：** 防止模型被告知使用 pixels 坐标但执行器按 normalized 变换，导致操作偏移。

### 16.4 终端代理宿主

**决策：** 终端作为 surrogate host，使用 `__CFBundleIdentifier` + 回退映射。

**原因：** CLI 无窗口，不能使用真实 frontmost gate。终端需要被排除在截图和隐藏之外，同时允许其 "frontmost" 状态不阻止操作。

### 16.5 拖拽动画 vs 瞬间移动

**决策：** 仅拖拽使用动画 (`animatedMove`)，click/scroll 使用瞬间移动。

**原因：** 中间动画帧触发 hover 状态，且在 decomposed mouseDown/moveMouse 路径上生成杂散 `.leftMouseDragged` 事件。但拖拽目标应用可能需要观察到拖拽中间位置（如滚动条、窗口调整）。
