
前言

Pretext 是一个纯 JavaScript/TypeScript 库，用于多行文本测量与布局。它快速、准确，并且支持各种你甚至没听说过的语言。支持渲染到 DOM、Canvas、SVG，未来还将支持服务端。

Pretext 绕过了对 DOM 测量的需求（例如 getBoundingClientRect、offsetHeight），这些操作会触发布局重排，这是浏览器中最昂贵的操作之一。它实现了自己的文本测量逻辑，以浏览器自身的字体引擎作为基准（一种非常适合 AI 的迭代方法）。

安装

```sh
npm install @chenglou/pretext
```

演示

克隆仓库，运行 bun install，然后运行 bun start，并在浏览器中打开 /demos 目录（不要带尾部斜杠。Bun 的开发服务器在这些路径上有 bug）。
或者，在线查看演示：chenglou.me/pretext。更多演示请访问 somnai-dreams.github.io/pretext-demos。

API

Pretext 服务于两种使用场景：

1. 测量段落高度 —— 完全无需触碰 DOM

```ts
import { prepare, layout } from '@chenglou/pretext'

const prepared = prepare('AGI 春天到了. بدأت الرحلة 🚀', '16px Inter')
const { height, lineCount } = layout(prepared, textWidth, 20) // 纯算术运算。没有 DOM 布局和重排！
```

prepare() 执行一次性工作：规范化空白字符、分割文本、应用粘合规则、使用 canvas 测量文本片段，并返回一个不透明的句柄。layout() 是之后的高频热路径：仅对缓存的宽度进行纯算术运算。对于相同的文本和配置，不要重新运行 prepare()，这会破坏其预计算的优势。例如，在窗口大小调整时，只需重新运行 layout()。

如果你想要类似 textarea 的文本行为，即普通空格、\t 制表符和 \n 硬换行保持可见，可以在 prepare() 中传入 { whiteSpace: 'pre-wrap' }：

```ts
const prepared = prepare(textareaValue, '16px Inter', { whiteSpace: 'pre-wrap' })
const { height } = layout(prepared, textareaWidth, 20)
```

根据当前已提交的基准测试快照：

· prepare() 处理共享的 500 段文本批次大约需要 19ms
· layout() 处理同一批次大约需要 0.09ms

我们支持你能想到的所有语言，包括表情符号和混合双向文本，并处理了特定的浏览器 quirks。

返回的高度是实现 Web UI 以下功能的关键：

· 精确的虚拟化/遮挡，无需猜测和缓存
· 高级的用户自定义布局：瀑布流、类似 JS 驱动的 flexbox 实现、无需 CSS 黑科技微调布局值等
· 开发时验证（尤其是在 AI 时代），例如检查按钮上的标签是否会换行溢出，且无需浏览器环境
· 当新文本加载并需要重新锚定滚动位置时，防止布局偏移

2. 手动布局段落中的每一行

将 prepare 替换为 prepareWithSegments，然后：

· 使用 layoutWithLines() 获取固定宽度下的所有行：

```ts
import { prepareWithSegments, layoutWithLines } from '@chenglou/pretext'

const prepared = prepareWithSegments('AGI 春天到了. بدأت الرحلة 🚀', '18px "Helvetica Neue"')
const { lines } = layoutWithLines(prepared, 320, 26) // 320px 最大宽度, 26px 行高
for (let i = 0; i < lines.length; i++) ctx.fillText(lines[i].text, 0, i * 26)
```

· 使用 walkLineRanges() 获取行宽和游标信息，而无需构建文本字符串：

```ts
let maxW = 0
walkLineRanges(prepared, 320, line => { if (line.width > maxW) maxW = line.width })
// maxW 现在是最宽行的宽度 —— 能够刚好容纳该文本的最紧凑容器宽度！这种多行"收缩包裹"效果一直是 Web 上所缺失的功能
```

· 使用 layoutNextLine() 在宽度动态变化时逐行处理文本：

```ts
let cursor = { segmentIndex: 0, graphemeIndex: 0 }
let y = 0

// 使文本环绕一个浮动图片：图片旁边的行较窄
while (true) {
  const width = y < image.bottom ? columnWidth - image.width : columnWidth
  const line = layoutNextLine(prepared, cursor, width)
  if (line === null) break
  ctx.fillText(line.text, 0, y)
  cursor = line.end
  y += 26
}
```

此用法支持渲染到 Canvas、SVG、WebGL，以及（最终将支持）服务端。

API 词汇表

用例 1 的 API：

```ts
prepare(text: string, font: string, options?: { whiteSpace?: 'normal' | 'pre-wrap' }): PreparedText // 一次性文本分析 + 测量过程，返回一个不透明值，传递给 `layout()`。请确保 `font` 与你为所测量文本声明的 CSS `font` 简写属性（如大小、粗细、样式、字体族）同步。`font` 的格式与你用于 `myCanvasContext.font = ...` 的格式相同，例如 `'16px Inter'`。
layout(prepared: PreparedText, maxWidth: number, lineHeight: number): { height: number, lineCount: number } // 根据给定的最大宽度和行高计算文本高度。请确保 `lineHeight` 与你为所测量文本声明的 CSS `line-height` 属性同步。
```

用例 2 的 API：

```ts
prepareWithSegments(text: string, font: string, options?: { whiteSpace?: 'normal' | 'pre-wrap' }): PreparedTextWithSegments // 与 `prepare()` 相同，但返回一个更丰富的结构，用于手动行布局需求
layoutWithLines(prepared: PreparedTextWithSegments, maxWidth: number, lineHeight: number): { height: number, lineCount: number, lines: LayoutLine[] } // 用于手动布局需求的高级 API。接受所有行的固定最大宽度。返回值与 `layout()` 类似，但额外返回行信息
walkLineRanges(prepared: PreparedTextWithSegments, maxWidth: number, onLine: (line: LayoutLineRange) => void): number // 用于手动布局需求的底层 API。接受所有行的固定最大宽度。对每一行调用一次 `onLine`，提供该行实际计算出的宽度以及起始/结束游标，而不构建行的文本字符串。这在某些场景下非常有用，例如，你想通过重复调用 walkLineRanges 并检查行数（从而检查高度）是否"理想"，来推测性地测试几个宽度和高度边界（例如二分查找一个合适的宽度值）。通过这种方式，你可以实现文本消息的收缩包裹和平衡文本布局。在 walkLineRanges 调用之后，你可以使用令人满意的最大宽度调用一次 layoutWithLines 来获取实际的行信息。
layoutNextLine(prepared: PreparedTextWithSegments, start: LayoutCursor, maxWidth: number): LayoutLine | null // 类似于迭代器的 API，用于为每一行设置不同的宽度进行布局！从 `start` 开始返回 LayoutLine，当段落结束时返回 `null`。将前一行的 `end` 游标作为下一次调用的 `start` 传入。
type LayoutLine = {
  text: string // 此行的完整文本内容，例如 'hello world'
  width: number // 此行的测量宽度，例如 87.5
  start: LayoutCursor // 在预处理片段/字素中的包含性起始游标
  end: LayoutCursor // 在预处理片段/字素中的排他性结束游标
}
type LayoutLineRange = {
  width: number // 此行的测量宽度，例如 87.5
  start: LayoutCursor // 在预处理片段/字素中的包含性起始游标
  end: LayoutCursor // 在预处理片段/字素中的排他性结束游标
}
type LayoutCursor = {
  segmentIndex: number // 在 prepareWithSegments 预处理后的富文本片段流中的片段索引
  graphemeIndex: number // 该片段内的字素索引；在片段边界处为 `0`
}
```

其他辅助函数：

```ts
clearCache(): void // 清除 Pretext 内部由 prepare() 和 prepareWithSegments() 使用的共享缓存。如果你的应用循环使用许多不同的字体或文本变体，并希望释放累积的缓存，此函数非常有用
setLocale(locale?: string): void // 可选（默认使用当前 locale）。为后续的 prepare() 和 prepareWithSegments() 设置 locale。内部也会调用 clearCache()。设置新的 locale 不会影响已有的 prepare() 和 prepareWithSegments() 状态（不会对它们进行修改）
```

注意事项

Pretext 并不打算（目前）成为一个完整的字体渲染引擎。它目前针对常见的文本设置：

· white-space: normal
· word-break: normal
· overflow-wrap: break-word
· line-break: auto
· 如果你传入 { whiteSpace: 'pre-wrap' }，普通空格、\t 制表符和 \n 硬换行将被保留，而不是被折叠。制表符遵循默认的浏览器样式 tab-size: 8。其他换行默认值保持不变：word-break: normal、overflow-wrap: break-word 和 line-break: auto。
· 在 macOS 上，system-ui 对于 layout() 的准确性来说是不安全的。请使用指定的字体。
· 由于默认目标包含 overflow-wrap: break-word，非常窄的宽度仍然可能在单词内部断开，但仅在字素边界处断开。

开发

请参阅 DEVELOPMENT.md 了解开发设置和命令。

致谢

Sebastian Markbage 在十年前首先埋下了 text-layout 的种子。他的设计 —— 使用 canvas 的 measureText 进行字形整形、使用 pdf.js 处理双向文本、流式换行 —— 为我们不断推进的架构提供了灵感。