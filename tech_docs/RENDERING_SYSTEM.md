---
title: Claude Code 渲染系统
aliases: [渲染系统, Rendering System]
series: Claude Code 架构解析
category: UI与可观测性
order: 18
tags:
  - claude-code
  - ui
  - react
  - ink
  - terminal
  - markdown-rendering
date: 2026-04-04
---

# Claude Code 渲染系统 (Rendering System) — 深度技术文档

## 目录

1. [架构概览](#1-架构概览)
2. [渲染管线全景](#2-渲染管线全景)
3. [Markdown 解析引擎](#3-markdown-解析引擎)
   - 3.1 [marked 词法分析器配置](#31-marked-词法分析器配置)
   - 3.2 [快速路径与 LRU Token 缓存](#32-快速路径与-lru-token-缓存)
   - 3.3 [formatToken — 逐 Token 转 ANSI](#33-formattoken--逐-token-转-ansi)
   - 3.4 [支持的 Markdown Token 类型全表](#34-支持的-markdown-token-类型全表)
4. [代码块语法高亮](#4-代码块语法高亮)
   - 4.1 [cli-highlight 异步懒加载](#41-cli-highlight-异步懒加载)
   - 4.2 [Suspense 降级策略](#42-suspense-降级策略)
5. [表格双模渲染](#5-表格双模渲染)
   - 5.1 [水平表格模式](#51-水平表格模式)
   - 5.2 [列宽自适应算法](#52-列宽自适应算法)
   - 5.3 [垂直 Key-Value 模式](#53-垂直-key-value-模式)
   - 5.4 [终端 resize 安全保护](#54-终端-resize-安全保护)
6. [超链接系统](#6-超链接系统)
   - 6.1 [OSC 8 终端超链接](#61-osc-8-终端超链接)
   - 6.2 [GitHub Issue 自动链接化](#62-github-issue-自动链接化)
7. [图片处理](#7-图片处理)
8. [ANSI 渲染层 — `<Ansi>` 组件](#8-ansi-渲染层--ansi-组件)
   - 8.1 [termio Parser 解析流程](#81-termio-parser-解析流程)
   - 8.2 [Span 合并优化](#82-span-合并优化)
   - 8.3 [样式属性映射](#83-样式属性映射)
9. [流式渲染优化 — `StreamingMarkdown`](#9-流式渲染优化--streamingmarkdown)
   - 9.1 [稳定前缀单调递增算法](#91-稳定前缀单调递增算法)
   - 9.2 [与 React StrictMode 的兼容](#92-与-react-strictmode-的兼容)
10. [消息路由与内容分发](#10-消息路由与内容分发)
    - 10.1 [Message 类型分发](#101-message-类型分发)
    - 10.2 [助手文本 vs 用户文本](#102-助手文本-vs-用户文本)
11. [辅助渲染组件](#11-辅助渲染组件)
12. [性能优化设计模式](#12-性能优化设计模式)
13. [设计模式总结与可借鉴点](#13-设计模式总结与可借鉴点)

---

## 1. 架构概览

Claude Code 是基于 **Ink (React for CLI)** 构建的终端 TUI 应用。所有富文本最终都转化为 **ANSI 转义序列**在终端中显示。渲染系统是一个精心设计的多层管线，核心由三大部分组成：

```
┌──────────────────────────────────────────────────────────────────┐
│              Markdown 解析层 (src/utils/markdown.ts)              │
│  configureMarked() — marked 库配置（禁用删除线等）                 │
│  formatToken()     — Token→ANSI 字符串的核心转换函数               │
│  applyMarkdown()   — 一步到位的便捷接口                           │
│  linkifyIssueReferences() — GitHub issue 自动链接化               │
└──────────────────────────────────────────────────────────────────┘
┌──────────────────────────────────────────────────────────────────┐
│              React 组件层 (src/components/)                       │
│  Markdown           — 主渲染组件，混合 ANSI+React 布局            │
│  StreamingMarkdown  — 流式增量渲染，稳定前缀算法                   │
│  MarkdownTable      — 表格专用组件，双模自适应                     │
│  HighlightedCode    — 文件片段语法高亮                            │
│  StructuredDiff     — 代码差异展示                                │
└──────────────────────────────────────────────────────────────────┘
┌──────────────────────────────────────────────────────────────────┐
│              终端输出层 (src/ink/)                                 │
│  <Ansi>       — ANSI 转义序列→Ink <Text>/<Link> 的桥接组件        │
│  termio.Parser — ANSI 序列解析器                                  │
│  wrapAnsi      — ANSI-aware 文本换行                              │
│  stringWidth   — 考虑 CJK/emoji 的字符串宽度计算                   │
└──────────────────────────────────────────────────────────────────┘
```

**核心设计理念**: Markdown 是唯一的富文本格式。所有 AI 输出都被当作 Markdown 处理，通过 `marked` 词法分析后，按 token 类型分别转换为带 ANSI 样式的字符串，最终通过 `<Ansi>` 组件在 Ink 终端渲染。表格是唯一例外——它需要独立的 React 组件做 flexbox 布局。

---

## 2. 渲染管线全景

一段 AI 回复文本从接收到显示，经历以下完整管线：

```
                         AI 回复文本
                              │
                              ▼
                   ┌─────────────────────┐
                   │ stripPromptXMLTags  │  去除提示词 XML 标签
                   └──────────┬──────────┘
                              ▼
                   ┌─────────────────────┐
                   │ hasMarkdownSyntax   │  正则快速检测 (#*`|[>-_~ 等)
                   └──────┬───────┬──────┘
                    有 MD │       │ 无 MD
                    语法  │       │ 语法
                          ▼       ▼
                   marked.lexer  构造单个 paragraph token
                          │       │
                          ▼       ▼
                   ┌─────────────────────┐
                   │  LRU Token Cache    │  hash key, 最多 500 条
                   └──────────┬──────────┘
                              ▼
                   ┌─────────────────────┐
                   │  Token 分流         │
                   └──────┬───────┬──────┘
              非 table    │       │  table
                          ▼       ▼
              formatToken()    <MarkdownTable>
                   │               │
                   ▼               ▼
           ANSI 字符串累积     React 组件布局
                   │               │
                   ▼               ▼
              <Ansi>          <Ansi>
                   │               │
                   ▼               ▼
              Ink <Text> / <Link>
                   │
                   ▼
              终端 ANSI 输出
```

**关键文件索引**:
- `src/components/Markdown.tsx` — 入口组件 + 缓存 + 流式渲染
- `src/utils/markdown.ts` — `formatToken()` 核心转换 + `applyMarkdown()` 便捷接口
- `src/components/MarkdownTable.tsx` — 表格双模渲染
- `src/ink/Ansi.tsx` — ANSI→React 桥接组件
- `src/utils/cliHighlight.ts` — 语法高亮异步加载
- `src/utils/hyperlink.ts` — OSC 8 超链接

---

## 3. Markdown 解析引擎

### 3.1 marked 词法分析器配置

Claude Code 使用 `marked` 库做 GFM (GitHub Flavored Markdown) 词法分析，但进行了一个重要定制：

```typescript
// src/utils/markdown.ts
export function configureMarked(): void {
  if (markedConfigured) return
  markedConfigured = true

  // 禁用删除线解析 — 模型经常用 ~ 表示"大约"
  // (如 ~100)，很少真正需要删除线格式
  marked.use({
    tokenizer: {
      del() {
        return undefined
      },
    },
  })
}
```

**设计考量**: AI 模型在代码/数学讨论中频繁使用 `~` 符号（如 `~100ms`、`O(~n)`），如果启用 `~~删除线~~` 解析会导致大量误触。禁用后 `~` 被当作普通文本处理。

### 3.2 快速路径与 LRU Token 缓存

这是渲染性能的核心优化之一，有两个层次：

**第一层：无 Markdown 语法快速路径**

```typescript
// src/components/Markdown.tsx
const MD_SYNTAX_RE = /[#*`|[>\-_~]|\n\n|^\d+\. |\n\d+\. /

function hasMarkdownSyntax(s: string): boolean {
  // 只检查前 500 字符 — Markdown 语法通常在开头（标题、代码围栏、列表）
  // 长工具输出的尾部大多是纯文本
  return MD_SYNTAX_RE.test(s.length > 500 ? s.slice(0, 500) : s)
}
```

如果文本不包含任何 Markdown 特征字符，直接构造一个合成的 paragraph token，**完全跳过 `marked.lexer`**（~3ms 的开销）：

```typescript
function cachedLexer(content: string): Token[] {
  if (!hasMarkdownSyntax(content)) {
    return [{
      type: 'paragraph',
      raw: content,
      text: content,
      tokens: [{ type: 'text', raw: content, text: content }]
    } as Token]
  }
  // ...
}
```

**第二层：基于 Content Hash 的 LRU 缓存**

对需要完整解析的内容，按 hash key 缓存 token 列表，最多保留 500 条，采用 `Map` 的插入序实现 LRU 淘汰：

```typescript
const TOKEN_CACHE_MAX = 500
const tokenCache = new Map<string, Token[]>()

function cachedLexer(content: string): Token[] {
  // ... 快速路径 ...

  const key = hashContent(content)
  const hit = tokenCache.get(key)
  if (hit) {
    // 提升到 MRU 位置 — 防止 FIFO 淘汰正在查看的消息
    tokenCache.delete(key)
    tokenCache.set(key, hit)
    return hit
  }

  const tokens = marked.lexer(content)
  if (tokenCache.size >= TOKEN_CACHE_MAX) {
    // 淘汰最老条目（Map 保持插入序）
    const first = tokenCache.keys().next().value
    if (first !== undefined) tokenCache.delete(first)
  }
  tokenCache.set(key, tokens)
  return tokens
}
```

**为什么用 hash key 而非原始内容？** 文档中明确注释：用原始内容做 key 会在长对话中（turn50→turn99）导致严重的 RSS 内存回归（#24180），因为 Map 键+值都持有完整内容引用。

**为什么快速路径不缓存？** 合成 token 只需一次对象分配，缓存反而会持有 4× 内容引用（raw/text 字段×2 层），得不偿失。

### 3.3 formatToken — 逐 Token 转 ANSI

`formatToken()` 是整个渲染系统的核心函数，它将 `marked` 的每个 token 递归地转换为带 ANSI 样式的字符串：

```typescript
// src/utils/markdown.ts
export function formatToken(
  token: Token,
  theme: ThemeName,       // 主题名（light/dark/...）
  listDepth = 0,          // 列表嵌套深度
  orderedListNumber: number | null = null,  // 有序列表当前编号
  parent: Token | null = null,    // 父 token（用于上下文感知）
  highlight: CliHighlight | null = null,  // 语法高亮器（可能未加载）
): string {
  switch (token.type) {
    // ... 约 20 种 token 类型的处理
  }
}
```

**上下文感知渲染的精妙之处**:

`parent` 参数使 `formatToken` 能根据上下文做不同处理。最典型的例子是 `text` token：

```typescript
case 'text':
  if (parent?.type === 'link') {
    // 在链接内部 — 不做 issue linkify，避免嵌套 OSC 8 序列
    // 嵌套的 OSC 8 中，终端以最内层为准，会覆盖链接的实际 href
    return token.text
  }
  if (parent?.type === 'list_item') {
    // 在列表项内部 — 添加列表符号前缀
    return `${orderedListNumber === null ? '-' : getListNumber(listDepth, orderedListNumber) + '.'} ${/* ... */}`
  }
  return linkifyIssueReferences(token.text)
```

### 3.4 支持的 Markdown Token 类型全表

| Token 类型 | ANSI 渲染效果 | 实现方式 | 备注 |
|---|---|---|---|
| `heading` (h1) | 粗体+斜体+下划线 | `chalk.bold.italic.underline()` | 后跟双换行 |
| `heading` (h2+) | 粗体 | `chalk.bold()` | 后跟双换行 |
| `strong` | **粗体** | `chalk.bold()` | 递归处理子 token |
| `em` | *斜体* | `chalk.italic()` | 递归处理子 token |
| `code` (围栏) | 语法高亮 | `cli-highlight` / 降级纯文本 | 不支持的语言 fallback |
| `codespan` (行内) | 主题色 | `color('permission', theme)` | 用权限色标记 |
| `blockquote` | 竖线前缀+斜体 | `chalk.dim(BAR)` + `chalk.italic()` | 每行加前缀 |
| `link` | OSC 8 可点击链接 | `createHyperlink()` | mailto: 降级为纯文本 |
| `image` | 输出 URL | `token.href` | 终端不渲染图像 |
| `list` | 嵌套缩进列表 | 递归 + `listDepth` 跟踪 | 支持有序/无序 |
| `list_item` | 缩进+符号 | `'  '.repeat(depth)` + 符号 | 多级编号格式变化 |
| `paragraph` | 正文段落 | 递归子 token + 换行 | — |
| `table` | 特殊处理 | 见 [§5 表格双模渲染](#5-表格双模渲染) | 两种渲染路径 |
| `hr` | `---` | 直接输出 | — |
| `space` / `br` | 换行符 | `\n` | — |
| `escape` | 转义字符 | `token.text` | `\)` → `)` 等 |
| `text` | 上下文感知 | 根据 parent 不同处理 | 见上方分析 |
| `del` / `html` / `def` | **不渲染** | 返回空字符串 | 被禁用/不支持 |

**有序列表编号格式随深度变化**:

```typescript
function getListNumber(listDepth: number, orderedListNumber: number): string {
  switch (listDepth) {
    case 0:
    case 1: return orderedListNumber.toString()    // 1. 2. 3.
    case 2: return numberToLetter(orderedListNumber) // a. b. c.
    case 3: return numberToRoman(orderedListNumber)  // i. ii. iii.
    default: return orderedListNumber.toString()     // 回到数字
  }
}
```

---

## 4. 代码块语法高亮

### 4.1 cli-highlight 异步懒加载

语法高亮使用 `cli-highlight`（底层 `highlight.js`），采用**异步懒加载**避免阻塞首屏：

```typescript
// src/utils/cliHighlight.ts
export type CliHighlight = {
  highlight: typeof import('cli-highlight').highlight
  supportsLanguage: typeof import('cli-highlight').supportsLanguage
}

let cliHighlightPromise: Promise<CliHighlight | null> | undefined

async function loadCliHighlight(): Promise<CliHighlight | null> {
  try {
    const cliHighlight = await import('cli-highlight')
    const highlightJs = await import('highlight.js')
    loadedGetLanguage = highlightJs.getLanguage
    return {
      highlight: cliHighlight.highlight,
      supportsLanguage: cliHighlight.supportsLanguage,
    }
  } catch {
    return null  // 加载失败静默降级
  }
}

// 全局单例 Promise — 多处共享
export function getCliHighlightPromise(): Promise<CliHighlight | null> {
  cliHighlightPromise ??= loadCliHighlight()
  return cliHighlightPromise
}
```

**设计亮点**: `highlight.js` 的 import 搭便车 — `cli-highlight` 内部已经加载了 `highlight.js`，第二次 `import('highlight.js')` 命中模块缓存，零额外开销。

### 4.2 Suspense 降级策略

`Markdown` 组件用 React `Suspense` 处理高亮器的异步加载：

```typescript
// src/components/Markdown.tsx
export function Markdown(props: Props): React.ReactNode {
  const settings = useSettings()

  // 用户可在设置中关闭语法高亮
  if (settings.syntaxHighlightingDisabled) {
    return <MarkdownBody {...props} highlight={null} />
  }

  // Suspense fallback: 先渲染无高亮版本（~50ms 后切换到有高亮版本）
  return (
    <Suspense fallback={<MarkdownBody {...props} highlight={null} />}>
      <MarkdownWithHighlight {...props} />
    </Suspense>
  )
}

function MarkdownWithHighlight(props: Props): React.ReactNode {
  const highlight = use(getCliHighlightPromise())  // React 19 的 use()
  return <MarkdownBody {...props} highlight={highlight} />
}
```

**渲染策略**: 首次渲染时，`cli-highlight` 可能还在加载。Suspense fallback 立即渲染 `highlight=null` 的版本（代码块显示为纯文本）。约 50ms 后异步加载完成，自动切换到语法高亮版本。后续渲染直接使用缓存的 Promise。

**在 `formatToken` 中的降级处理**:

```typescript
case 'code': {
  if (!highlight) {
    return token.text + EOL  // 无高亮器 → 原始文本
  }
  let language = 'plaintext'
  if (token.lang) {
    if (highlight.supportsLanguage(token.lang)) {
      language = token.lang
    } else {
      logForDebugging(
        `Language not supported, falling back to plaintext: ${token.lang}`
      )
    }
  }
  return highlight.highlight(token.text, { language }) + EOL
}
```

---

## 5. 表格双模渲染

表格是渲染系统中最复杂的部分。它是唯一一个不走 ANSI 字符串累积路径，而是使用独立 React 组件做布局的格式。原因是表格需要精确的列宽计算和 flexbox 对齐，这在纯字符串拼接中无法优雅实现。

### 5.1 水平表格模式

正常终端宽度下，渲染为 Unicode box-drawing 字符的表格：

```
┌──────────┬──────────────┬────────┐
│  Name    │  Description │  Size  │
├──────────┼──────────────┼────────┤
│  foo.ts  │  Main entry  │  1.2K  │
├──────────┼──────────────┼────────┤
│  bar.ts  │  Utils       │  456B  │
└──────────┴──────────────┴────────┘
```

边框字符使用完整的 box-drawing 集合：

```typescript
function renderBorderLine(type: 'top' | 'middle' | 'bottom'): string {
  const [left, mid, cross, right] = {
    top:    ['┌', '─', '┬', '┐'],
    middle: ['├', '─', '┼', '┤'],
    bottom: ['└', '─', '┴', '┘']
  }[type]
  // ...
}
```

### 5.2 列宽自适应算法

列宽计算是一个三阶段过程：

**阶段 1: 计算每列的最小宽度和理想宽度**

```typescript
// 最小宽度 = max(最长单词宽度, 3)
function getMinWidth(tokens): number {
  const text = getPlainText(tokens)
  const words = text.split(/\s+/).filter(w => w.length > 0)
  return Math.max(...words.map(w => stringWidth(w)), MIN_COLUMN_WIDTH)
}

// 理想宽度 = max(内容总宽度, 3)
function getIdealWidth(tokens): number {
  return Math.max(stringWidth(getPlainText(tokens)), MIN_COLUMN_WIDTH)
}
```

**阶段 2: 根据可用空间选择策略**

```typescript
const borderOverhead = 1 + numCols * 3  // │ + (2 padding + 1 border) per col
const availableWidth = terminalWidth - borderOverhead - SAFETY_MARGIN

if (totalIdeal <= availableWidth) {
  // 情况 A: 全部适配 → 使用理想宽度
  columnWidths = idealWidths
} else if (totalMin <= availableWidth) {
  // 情况 B: 需要压缩 → 每列给最小宽度，剩余空间按溢出比例分配
  const extraSpace = availableWidth - totalMin
  const overflows = idealWidths.map((ideal, i) => ideal - minWidths[i])
  const totalOverflow = overflows.reduce((sum, o) => sum + o, 0)
  columnWidths = minWidths.map((min, i) => {
    const extra = Math.floor(overflows[i] / totalOverflow * extraSpace)
    return min + extra
  })
} else {
  // 情况 C: 终端太窄 → 按比例缩放，允许断词
  needsHardWrap = true
  const scaleFactor = availableWidth / totalMin
  columnWidths = minWidths.map(w => Math.max(Math.floor(w * scaleFactor), MIN_COLUMN_WIDTH))
}
```

**阶段 3: 判断是否需要切换垂直模式**

```typescript
const MAX_ROW_LINES = 4

const maxRowLines = calculateMaxRowLines()
const useVerticalFormat = maxRowLines > MAX_ROW_LINES
```

### 5.3 垂直 Key-Value 模式

当列内容换行导致行高超过 4 行时，自动切换到 key-value 格式：

```
Name: foo.ts
  Description: Main entry point for the application
  Size: 1.2K
────────────────────────────────
Name: bar.ts
  Description: Utility functions
  Size: 456B
```

实现中有一个精妙的**两段式换行**：第一行因为有 label 前缀所以较窄，后续行可以用更宽的宽度：

```typescript
const firstLineWidth = terminalWidth - stringWidth(label) - 3
const subsequentLineWidth = terminalWidth - wrapIndent.length - 1

const firstPassLines = wrapText(value, Math.max(firstLineWidth, 10))
const firstLine = firstPassLines[0] || ''

if (firstPassLines.length <= 1 || subsequentLineWidth <= firstLineWidth) {
  wrappedValue = firstPassLines
} else {
  // 剩余文本用更宽的宽度重新换行
  const remainingText = firstPassLines.slice(1).map(l => l.trim()).join(' ')
  const rewrapped = wrapText(remainingText, subsequentLineWidth)
  wrappedValue = [firstLine, ...rewrapped]
}
```

### 5.4 终端 resize 安全保护

表格渲染完成后，会做一次**安全检查**防止终端 resize 导致的渲染溢出：

```typescript
const SAFETY_MARGIN = 4

const maxLineWidth = Math.max(
  ...tableLines.map(line => stringWidth(stripAnsi(line)))
)

// 如果任何行超过终端宽度（减去安全边距），回退到垂直格式
if (maxLineWidth > terminalWidth - SAFETY_MARGIN) {
  return <Ansi>{renderVerticalFormat()}</Ansi>
}
```

**为什么需要 SAFETY_MARGIN?** 注释解释：消息缩进（如点号前缀）占用额外空间，加上终端 resize 时计算可能基于旧宽度。没有足够边距时，Ink 的 clip 在交替帧上截断不同，导致无限闪烁循环。

---

## 6. 超链接系统

### 6.1 OSC 8 终端超链接

支持 OSC 8 标准的终端（iTerm2, Windows Terminal, Hyper 等）中，链接显示为可点击的蓝色文本：

```typescript
// src/utils/hyperlink.ts
export const OSC8_START = '\x1b]8;;'
export const OSC8_END = '\x07'   // BEL 终止符，比 ST (\x1b\\) 兼容性更好

export function createHyperlink(
  url: string,
  content?: string,
): string {
  const hasSupport = supportsHyperlinks()
  if (!hasSupport) {
    return url  // 不支持时直接显示 URL
  }

  const displayText = content ?? url
  // 使用基本 ANSI 蓝色 — RGB 色彩不能和 OSC 8 配合 wrap-ansi
  const coloredText = chalk.blue(displayText)
  return `${OSC8_START}${url}${OSC8_END}${coloredText}${OSC8_START}${OSC8_END}`
}
```

**在 `formatToken` 中的链接处理**:

```typescript
case 'link': {
  // 防止 mailto: 链接被渲染为可点击链接
  if (token.href.startsWith('mailto:')) {
    const email = token.href.replace(/^mailto:/, '')
    return email
  }

  const linkText = (token.tokens ?? [])
    .map(_ => formatToken(_, theme, 0, null, token, highlight))
    .join('')
  const plainLinkText = stripAnsi(linkText)

  // 有意义的显示文本（与 URL 不同）→ 显示为可点击超链接
  if (plainLinkText && plainLinkText !== token.href) {
    return createHyperlink(token.href, linkText)
  }
  // 显示文本和 URL 相同 → 直接显示 URL
  return createHyperlink(token.href)
}
```

### 6.2 GitHub Issue 自动链接化

纯文本中的 `owner/repo#123` 格式自动转为 GitHub 链接：

```typescript
// 匹配 owner/repo#NNN，但不匹配 docs.github.io/guide#42
// Owner 不允许点号（GitHub 用户名只有字母数字和连字符）
// Repo 允许点号（如 cc.kurs.web）
const ISSUE_REF_PATTERN =
  /(^|[^\w./-])([A-Za-z0-9][\w-]*\/[A-Za-z0-9][\w.-]*)#(\d+)\b/g

function linkifyIssueReferences(text: string): string {
  if (!supportsHyperlinks()) return text

  return text.replace(
    ISSUE_REF_PATTERN,
    (_match, prefix, repo, num) =>
      prefix + createHyperlink(
        `https://github.com/${repo}/issues/${num}`,
        `${repo}#${num}`,
      ),
  )
}
```

**设计决策**: 早期版本也会链接化裸 `#NNN` 引用（通过猜测当前仓库），但这在讨论其他仓库时经常出错，因此只保留了完全限定的 `owner/repo#NNN` 格式。

---

## 7. 图片处理

Claude Code 作为终端应用，**不支持内联图像像素渲染**。图片以两种方式处理：

**Markdown 内的图片引用**:

```typescript
// src/utils/markdown.ts — formatToken
case 'image':
  return token.href  // 只输出 URL 文本
```

**用户上传的图片附件**:

```typescript
// src/components/messages/UserImageMessage.tsx
export function UserImageMessage({ imageId, addMargin }) {
  const label = imageId ? `[Image #${imageId}]` : "[Image]"
  const imagePath = imageId ? getStoredImagePath(imageId) : null

  // 如果终端支持超链接且图片有本地存储，显示为可点击的 file:// 链接
  const content = imagePath && supportsHyperlinks()
    ? <Link url={pathToFileURL(imagePath).href}><Text>{label}</Text></Link>
    : <Text>{label}</Text>

  // 图片作为新 turn 开头时需要 margin，否则用 MessageResponse 连接样式
  return addMargin
    ? <Box marginTop={1}>{content}</Box>
    : <MessageResponse>{content}</MessageResponse>
}
```

**设计考量**: 虽然部分终端（iTerm2 等）支持 sixel/iterm2 inline images，但 Claude Code 选择不实现——原因可能是：(1) 兼容性差异大，(2) 终端图片渲染质量参差不齐，(3) 对 CLI 交互的核心场景（代码/文本）价值有限。

---

## 8. ANSI 渲染层 — `<Ansi>` 组件

`<Ansi>` 是连接 "ANSI 字符串世界" 和 "Ink React 组件世界" 的桥梁。所有 `formatToken` 生成的带样式字符串最终都通过它渲染。

### 8.1 termio Parser 解析流程

```typescript
// src/ink/Ansi.tsx
function parseToSpans(input: string): Span[] {
  const parser = new Parser()   // termio 的 ANSI 解析器
  const actions = parser.feed(input)
  const spans: Span[] = []
  let currentHyperlink: string | undefined

  for (const action of actions) {
    if (action.type === 'link') {
      // OSC 8 超链接的 start/end 追踪
      if (action.action.type === 'start') {
        currentHyperlink = action.action.url
      } else {
        currentHyperlink = undefined
      }
      continue
    }

    if (action.type === 'text') {
      const text = action.graphemes.map(g => g.value).join('')
      if (!text) continue
      const props = textStyleToSpanProps(action.style)
      if (currentHyperlink) {
        props.hyperlink = currentHyperlink
      }

      // 合并相邻同样式 span
      const lastSpan = spans[spans.length - 1]
      if (lastSpan && propsEqual(lastSpan.props, props)) {
        lastSpan.text += text
      } else {
        spans.push({ text, props })
      }
    }
  }
  return spans
}
```

### 8.2 Span 合并优化

相邻的同样式文本段会被合并为单个 Span，减少 React 节点数量。这对长文本输出特别重要。判断两个 Span 是否可合并：

```typescript
function propsEqual(a: SpanProps, b: SpanProps): boolean {
  return a.color === b.color
    && a.backgroundColor === b.backgroundColor
    && a.bold === b.bold
    && a.dim === b.dim
    && a.italic === b.italic
    && a.underline === b.underline
    && a.strikethrough === b.strikethrough
    && a.inverse === b.inverse
    && a.hyperlink === b.hyperlink
}
```

### 8.3 样式属性映射

`<Ansi>` 组件支持将 termio 的颜色系统完整映射到 Ink：

```typescript
// termio Named Color → Ink ansi: 格式
const NAMED_COLOR_MAP: Record<NamedColor, string> = {
  black: 'ansi:black',
  red: 'ansi:red',
  // ... 16 种命名色（含 bright 变体）
  brightWhite: 'ansi:whiteBright'
}

function colorToString(color: TermioColor): Color | undefined {
  switch (color.type) {
    case 'named':   return NAMED_COLOR_MAP[color.name]
    case 'indexed':  return `ansi256(${color.index})`     // 256 色
    case 'rgb':      return `rgb(${color.r},${color.g},${color.b})`  // 真彩色
    case 'default':  return undefined
  }
}
```

**bold/dim 互斥处理**: 终端中 bold 和 dim 是互斥的（因为它们都影响字重），`StyledText` 组件显式处理这种优先级：

```typescript
function StyledText({ bold, dim, children, ...rest }) {
  // dim 优先于 bold（终端视两者互斥）
  if (dim) return <Text {...rest} dim>{children}</Text>
  if (bold) return <Text {...rest} bold>{children}</Text>
  return <Text {...rest}>{children}</Text>
}
```

**`<Ansi>` 组件的优化快速路径**:

```typescript
export const Ansi = React.memo(function Ansi({ children, dimColor }) {
  // 快速路径 1: 空字符串
  if (children === '') return null

  const spans = parseToSpans(children)

  // 快速路径 2: 无 span
  if (spans.length === 0) return null

  // 快速路径 3: 单个无样式 span → 直接 <Text>
  if (spans.length === 1 && !hasAnyProps(spans[0].props)) {
    return dimColor ? <Text dim>{spans[0].text}</Text> : <Text>{spans[0].text}</Text>
  }

  // 完整路径: 遍历 span，生成 <StyledText> 和 <Link>
  const content = spans.map((span, i) => {
    if (span.props.hyperlink) {
      return <Link key={i} url={span.props.hyperlink}>
        <StyledText {...span.props}>{span.text}</StyledText>
      </Link>
    }
    return hasAnyTextProps(span.props)
      ? <StyledText key={i} {...span.props}>{span.text}</StyledText>
      : span.text  // 无样式文本直接作为字符串子节点
  })

  return dimColor ? <Text dim>{content}</Text> : <Text>{content}</Text>
})
```

`React.memo` 包裹确保父组件重渲时，如果 ANSI 字符串未变则跳过。

---

## 9. 流式渲染优化 — `StreamingMarkdown`

当 AI 正在输出回复时，文本是逐渐增长的。`StreamingMarkdown` 组件实现了高效的增量解析。

### 9.1 稳定前缀单调递增算法

核心思想：将流式文本分为"稳定前缀"和"不稳定后缀"两部分。已完成的 Markdown block 不会再变，无需重新解析。

```typescript
// src/components/Markdown.tsx
export function StreamingMarkdown({ children }: StreamingProps): React.ReactNode {
  'use no memo'  // 退出 React Compiler 自动 memo — 算法需要 ref 读写
  configureMarked()

  const stripped = stripPromptXMLTags(children)
  const stablePrefixRef = useRef('')

  // 防御性重置：如果新文本不以旧前缀开头，说明内容被替换
  if (!stripped.startsWith(stablePrefixRef.current)) {
    stablePrefixRef.current = ''
  }

  // 只从当前边界开始 lex — O(不稳定长度) 而非 O(全文长度)
  const boundary = stablePrefixRef.current.length
  const tokens = marked.lexer(stripped.substring(boundary))

  // 最后一个非 space token 是正在增长的 block
  // 它之前的 token 都已经完成
  let lastContentIdx = tokens.length - 1
  while (lastContentIdx >= 0 && tokens[lastContentIdx].type === 'space') {
    lastContentIdx--
  }

  // 推进稳定边界
  let advance = 0
  for (let i = 0; i < lastContentIdx; i++) {
    advance += tokens[i].raw.length
  }
  if (advance > 0) {
    stablePrefixRef.current = stripped.substring(0, boundary + advance)
  }

  const stablePrefix = stablePrefixRef.current
  const unstableSuffix = stripped.substring(stablePrefix.length)

  // stablePrefix 在 <Markdown> 内部被 useMemo 缓存，不会重新解析
  return (
    <Box flexDirection="column" gap={1}>
      {stablePrefix && <Markdown>{stablePrefix}</Markdown>}
      {unstableSuffix && <Markdown>{unstableSuffix}</Markdown>}
    </Box>
  )
}
```

**算法可视化**:

```
时刻 T1: "# Hello\n\nThis is a paragr"
         ┌──────────┐ ┌─────────────────┐
         │  稳定     │ │   不稳定          │
         │ # Hello  │ │ This is a paragr │ ← 正在增长
         └──────────┘ └─────────────────┘

时刻 T2: "# Hello\n\nThis is a paragraph.\n\n```python\nprint("
         ┌────────────────────────────────────┐ ┌────────────────────┐
         │            稳定                      │ │      不稳定         │
         │ # Hello\n\nThis is a paragraph.     │ │ ```python\nprint(  │
         └────────────────────────────────────┘ └────────────────────┘
         └─ 只渲染一次 ─────────────────────────┘ └─ 每次 delta 重新 lex ─┘
```

### 9.2 与 React StrictMode 的兼容

组件在 render 期间读写 `stablePrefixRef.current`，这在 StrictMode 下会被双重调用。但因为边界是**单调递增**的（只会前进不会后退），所以双重调用是幂等的 — 第二次执行只是重新设置同样的值。

`'use no memo'` 指令退出 React Compiler 的自动 memoization，因为 Compiler 无法证明 ref 读写的安全性，自动 memo 会缓存旧的边界值导致算法失效。

---

## 10. 消息路由与内容分发

### 10.1 Message 类型分发

`src/components/Message.tsx` 是消息渲染的入口，根据 `message.type` 分发到不同组件：

```
message.type === 'assistant'  → AssistantMessageBlock
                                  ├── AssistantTextMessage (文本)  → <Markdown>
                                  ├── AssistantToolUseMessage (工具调用)
                                  └── AssistantThinkingMessage (思考)
message.type === 'user'       → UserPromptMessage → <HighlightedThinkingText>
message.type === 'attachment'  → AttachmentMessage (文件/PDF/图片)
message.type === 'image'       → UserImageMessage → [Image #N] 链接
```

### 10.2 助手文本 vs 用户文本

一个重要的区分：

- **助手文本** (`AssistantTextMessage`) 使用 `<Markdown>` 组件渲染，支持完整的 Markdown 格式
- **用户文本** (`UserPromptMessage`) 使用 `<HighlightedThinkingText>` 渲染，**不经过 Markdown 解析**

```typescript
// src/components/messages/AssistantTextMessage.tsx
<Box flexDirection="column"><Markdown>{text}</Markdown></Box>

// src/components/messages/UserPromptMessage.tsx
// 用户输入不走 Markdown 管线
```

**设计原因**: 用户输入的文本可能包含大量 Markdown 特征字符（如代码片段），如果当作 Markdown 渲染会产生意外格式。

**流式文本的渲染切换**:

```typescript
// src/components/Messages.tsx 中的逻辑
// 正在流式输出时用 StreamingMarkdown
{streamingText && <StreamingMarkdown>{streamingText}</StreamingMarkdown>}
// 输出完成后切换为静态 Markdown
{!streamingText && text && <Markdown>{text}</Markdown>}
```

---

## 11. 辅助渲染组件

除了核心 Markdown 渲染管线，还有几个专用渲染组件：

| 组件 | 文件 | 用途 |
|---|---|---|
| `HighlightedCode` | `src/components/HighlightedCode.tsx` | 文件片段的语法高亮展示（带行号 gutter），独立于 Markdown 代码块 |
| `StructuredDiff` | `src/components/StructuredDiff.tsx` | 代码差异的结构化展示，ANSI-aware 切片 |
| `RawAnsi` | `src/ink/components/RawAnsi.tsx` | 直接输出原始 ANSI 字符串（用于 shell 输出等） |
| `OutputLine` | `src/components/shell/OutputLine.tsx` | Shell 命令输出的 ANSI 渲染，带截断辅助 |
| `PreviewBox` | `src/components/permissions/.../PreviewBox.tsx` | 权限 UI 中的 Markdown 预览，使用 `applyMarkdown()` |

**`applyMarkdown()` 便捷接口**:

```typescript
// src/utils/markdown.ts
export function applyMarkdown(
  content: string,
  theme: ThemeName,
  highlight: CliHighlight | null = null,
): string {
  configureMarked()
  return marked
    .lexer(stripPromptXMLTags(content))
    .map(_ => formatToken(_, theme, 0, null, null, highlight))
    .join('')
    .trim()
}
```

这个函数直接返回 ANSI 字符串（不经过 React 组件），用于不需要表格布局的简单场景，如权限确认对话框的 Markdown 预览。

---

## 12. 性能优化设计模式

渲染系统中使用了多种性能优化技巧，值得整理：

### 12.1 三级缓存策略

| 层次 | 位置 | 缓存对象 | 策略 | 容量 |
|---|---|---|---|---|
| L1 | `hasMarkdownSyntax` | 解析决策 | 正则快速路径跳过 | 无限（无状态） |
| L2 | `cachedLexer` | Token 列表 | Content Hash + LRU | 500 条 |
| L3 | React memo | 组件输出 | `React.memo` + `useMemo` | 按组件树 |

### 12.2 React Compiler 集成

所有组件都使用了 React Compiler 的 `_c()` 调用做自动 memoization：

```typescript
const $ = _c(12)  // 分配 12 个缓存槽位
// ...
if ($[0] !== children || $[1] !== dimColor) {
  // 依赖变化 → 重新计算
  t1 = /* ... */
  $[0] = children
  $[1] = dimColor
  $[2] = t1
} else {
  // 依赖未变 → 复用缓存
  t1 = $[2]
}
```

只有 `StreamingMarkdown` 因为 render-time ref mutation 退出了 Compiler（`'use no memo'`）。

### 12.3 Module-level Cache vs useMemo

Token cache 故意放在模块级而非组件 `useMemo` 中：

> `useMemo` doesn't survive unmount→remount, so scrolling back to a previously-visible message re-parses. Messages are immutable in history; same content → same tokens.

虚拟滚动中，向上滚动回到之前的消息会导致组件 unmount→remount，`useMemo` 的缓存在 unmount 时丢失。模块级缓存跨越组件生命周期，避免了这个问题。

### 12.4 ANSI 字符串 vs React 组件

渲染系统刻意选择 "ANSI 字符串优先" 策略：大部分内容先转为 ANSI 字符串再喂给 `<Ansi>` 组件。只有表格需要 React 组件布局。

**优势**: 字符串拼接远比 React 组件创建+协调快；一个 `<Ansi>` 节点替代了可能数十个 `<Text>` 节点的嵌套树。

---

## 13. 设计模式总结与可借鉴点

### 13.1 "Markdown 作为唯一富文本格式" 模式

**做法**: 不发明自定义格式，直接复用 Markdown 作为 AI 输出的标准格式。
**优势**: AI 模型天然擅长生成 Markdown；用户已熟悉 Markdown 语法；工具链成熟（marked, highlight.js 等）。
**可借鉴**: 任何 AI 工具的输出格式选择，Markdown 是最自然的选择。

### 13.2 "Token 分流" 模式

**做法**: 大部分 token 走字符串拼接快速路径，只有表格等复杂格式走 React 组件路径。
**优势**: 在不牺牲表格渲染质量的前提下，最大化简单内容的渲染性能。
**可借鉴**: 混合渲染策略 — 用简单方式处理 80% 的场景，用复杂方式处理 20% 的特殊场景。

### 13.3 "稳定前缀" 增量解析模式

**做法**: 流式文本中，只对最后一个未完成的 block 重新解析，已完成部分永不重新处理。
**优势**: 将 O(n) 的重复解析降为 O(delta)。
**可借鉴**: 任何需要处理增量输入的场景（实时编辑器、日志查看器等）。

### 13.4 "异步懒加载 + Suspense 降级" 模式

**做法**: 重型依赖（highlight.js）异步加载，首屏先渲染降级版本。
**优势**: 首屏 ~50ms 加速，用户几乎无感知。
**可借鉴**: 任何非首屏必需的重型依赖都应该异步加载，提供即时降级显示。

### 13.5 "Content Hash + LRU" 缓存模式

**做法**: 用内容 hash 而非原始内容做缓存 key，LRU 淘汰控制内存。
**优势**: 避免字符串引用保留导致的内存泄漏，同时保证 hash 碰撞概率极低。
**可借鉴**: 任何以大字符串为 key 的缓存都应该考虑 hash 化。

### 13.6 "终端能力检测 + 渐进增强" 模式

**做法**: 检测终端是否支持超链接（OSC 8），支持则显示可点击链接，否则显示纯 URL。
**优势**: 在任何终端都能工作，在高级终端提供更好体验。
**可借鉴**: 终端应用应该探测能力并渐进增强，而非假设所有终端相同。
