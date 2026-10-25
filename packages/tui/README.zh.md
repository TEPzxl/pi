> 译注：本文是 [README.md](README.md) 的简体中文译文，内容可能滞后于英文原文；所有代码示例均保持原样。

<a id="earendil-workspi-tui"></a>
# @earendil-works/pi-tui

极简终端 UI 框架，通过差分渲染（只更新变化区域）和同步输出，为交互式 CLI 应用提供无闪烁的显示效果。

<a id="features"></a>
## 特性

- **可互换的渲染器**：主屏幕与备用屏幕缓冲区实现共用 `TUI` 接口
- **差分渲染**：只更新发生变化的行或视口行
- **应用管理的滚动**：备用屏幕缓冲区中的视口支持鼠标、触控板和键盘导航
- **同步输出**：使用 CSI 2026 实现原子化屏幕更新（无闪烁）
- **Bracketed Paste Mode**：正确处理大段粘贴，并为超过 10 行的粘贴内容添加标记
- **基于组件**：简单的 Component 接口，提供 render() 方法
- **主题支持**：组件接受主题接口，以便自定义样式
- **内置组件**：Text、TruncatedText、Input、Editor、Markdown、Loader、SelectList、SettingsList、MouseRegion、Spacer、Image、Box、Container、VStack、HStack、ScrollView
- **内联图片**：在支持 Kitty 或 iTerm2 图形协议的终端中渲染图片
- **自动补全支持**：文件路径和斜杠命令

<a id="quick-start"></a>
## 快速开始

```typescript
import { type TUI, Text, Editor, ProcessTerminal, TuiMainScreen, matchesKey } from "@earendil-works/pi-tui";

// Create terminal
const terminal = new ProcessTerminal();

// Create the default main-screen renderer through the shared TUI interface
const tui: TUI = new TuiMainScreen(terminal);

// Add components
tui.addChild(new Text("Welcome to my app!"));

import { defaultEditorTheme as editorTheme } from './test/test-themes.ts';
const editor = new Editor(tui, editorTheme);
editor.onSubmit = (text) => {
  console.log("Submitted:", text);
  tui.addChild(new Text(`You said: ${text}`));
};
tui.addChild(editor);

// Focus the editor so it receives keyboard input
tui.setFocus(editor);

// In raw mode Ctrl+C doesn't send SIGINT — intercept it here to allow exit
tui.addInputListener((data) => {
  if (matchesKey(data, 'ctrl+c')) {
    tui.stop();
    process.exit(0);
  }
});

// Start
tui.start();
```

<a id="core-api"></a>
## 核心 API

<a id="tui-interface-and-renderers"></a>
### TUI 接口与渲染器

`TUI` 是用于组件管理、焦点、覆盖层、输入、生命周期、终端查询和渲染的共享接口。只需在构建应用时选择具体渲染器：

- `TuiMainScreen` 在终端的主屏幕缓冲区中渲染，并保留终端回滚历史。
- `TuiAltScreen` 在终端的备用屏幕缓冲区中渲染固定高度的视口，由应用管理滚动。停止时，它会恢复主屏幕缓冲区，并打印完整的最终文档。

```typescript
import { type TUI, TuiAltScreen, TuiMainScreen } from "@earendil-works/pi-tui";

const tui: TUI = new TuiMainScreen(terminal);
// To use an application-owned viewport in the alternate terminal buffer instead:
// const tui: TUI = new TuiAltScreen(terminal);

tui.addChild(component);
tui.removeChild(component);
tui.start();
tui.stop();
tui.requestRender(); // Request a re-render

// Global debug key handler (Shift+Ctrl+D)
tui.onDebug = () => console.log("Debug triggered");
```

<a id="colors-and-terminal-styles"></a>
### 颜色与终端样式

颜色是可在终端渲染前进行转换或混合的值：

```typescript
import {
  colorToRgb,
  foregroundAnsi,
  getTerminalColorMode,
  mixColors,
  parseColor,
  rgbColor,
  styleText,
} from "@earendil-works/pi-tui";

const accent = parseColor("oklch(70% 0.12 220)");
const background = parseColor("#20242a");
const foreground = mixColors(accent, background, 0.2);

const text = styleText(
  "Ready",
  { fg: foreground, bg: background, bold: true },
  getTerminalColorMode(),
);
```

`Color` 可以是索引式 ANSI 颜色、sRGB 颜色或 OKLCH 颜色。每种颜色都能转换为 sRGB，因此 `mixColors()` 等颜色运算始终可用。索引 0–15 遵循用户的终端调色板，因此其 sRGB 值是近似值。`styleText()` 会根据请求的终端模式，将颜色转换为 truecolor 或 256 色输出。

`parseColor()` 也接受 OKHSL，例如 `okhsl(250 60% 55%)`；`okhslColor()` 可在代码中构建这种颜色，`colorToOkhsl()` 则可读取任意颜色的 OKHSL 通道。OKHSL 饱和度以 sRGB 色域在该色相和明度下允许的最大饱和度为基准，因此每个值都在色域内，且不同色相下相同的饱和度看起来具有相同的鲜艳程度。OKHSL 颜色在创建时会转换为 sRGB。

转换结果不会缓存。OKLCH 颜色，尤其是超出 sRGB 色域的颜色，转换开销比 sRGB 或索引式颜色更大。对于每次渲染都会使用的颜色，应转换一次并复用结果：

```typescript
const { r, g, b } = colorToRgb(mixColors(accent, background, 0.2));
const foreground = rgbColor(r, g, b); // cheap to render repeatedly
const foregroundCode = foregroundAnsi(foreground, getTerminalColorMode());
```

<a id="alternate-screen-viewport-layouts"></a>
### 备用屏幕缓冲区的视口布局

`TuiAltScreen` 可以渲染显式指定的终端高度布局。`VStack` 和 `HStack` 分配受约束的区域，而 `ScrollView` 管理某一区域的滚动。这些语义有意不在 `TuiMainScreen` 中提供，因为其中的回滚历史由终端管理。

```typescript
import {
  Container,
  isViewportTUI,
  ScrollView,
  Text,
  VStack,
} from "@earendil-works/pi-tui";

const transcript = new Container();
transcript.addChild(new Text("History"));

const editorAndFooter = new VStack([
  editor,
  new Text("status"),
]);

if (isViewportTUI(tui)) {
  tui.setLayoutRoot(new VStack([
    {
      component: new ScrollView(transcript, {
        follow: "end",
        primary: true,
        overscroll: "chain",
      }),
      basis: 0,
      grow: 1,
      minSize: 1,
    },
    {
      component: editorAndFooter,
      basis: "auto",
      shrink: 1,
      minSize: 1,
    },
  ]));
}
```

Stack 条目支持 `basis`、`grow`、`shrink`、`minSize`、`maxSize`，以及用于响应式显示的 `visible` 回调。鼠标滚轮输入作用于指针下的滚动视图，默认会将未消耗的滚动增量沿链传递给外层滚动视图。主滚动视图接收备用屏幕缓冲区模式下的键盘导航操作，以及不可滚动区域上的滚轮输入。它还可以在 OSC 133 语义提示符标记之间跳转，与常见的终端提示符导航快捷键行为一致。按 `Ctrl+Shift+F` 可打开或关闭其带边框的搜索面板。面板会显示配置的上一个/下一个快捷键，并提供可点击的箭头控件；默认使用 `Enter`/`Ctrl+G` 和 `Shift+Enter`/`Ctrl+Shift+G` 在匹配项之间移动，`Escape` 也可关闭搜索。`TuiAltScreenOptions.searchMatchStyle` 和 `searchCurrentMatchStyle` 用于自定义匹配项高亮，而 `searchNavigationButtonStyle` 为每个箭头按钮设置样式，并接收其悬停状态。当设置了 `follow: "end"` 的主滚动视图滚动到远离末尾的位置时，`TuiAltScreenOptions.scrollToEndIndicator` 会在该视图最后一行的中央渲染一个可点击的标签；点击它即可恢复跟随末尾。

每次请求帧时都会重新构建布局几何信息。有状态组件会保留，其现有的渲染行缓存仍然有效。直接对这些布局组件调用 `render(width)` 会生成不受高度约束的文档，备用屏幕缓冲区模式恢复主屏幕时也使用该文档。

<a id="overlays"></a>
### 覆盖层

覆盖层将组件渲染在现有内容之上，而不替换原有内容。适用于对话框、菜单和模态 UI。

```typescript
// Show overlay with default options (centered, max 80 cols)
const handle = tui.showOverlay(component);

// Show overlay with custom positioning and sizing
// Values can be numbers (absolute) or percentage strings (e.g., "50%")
const handle = tui.showOverlay(component, {
  // Sizing
  width: 60,              // Fixed width in columns
  width: "80%",           // Width as percentage of terminal
  minWidth: 40,           // Minimum width floor
  maxHeight: 20,          // Maximum height in rows
  maxHeight: "50%",       // Maximum height as percentage of terminal

  // Anchor-based positioning (default: 'center')
  anchor: 'bottom-right', // Position relative to anchor point
  offsetX: 2,             // Horizontal offset from anchor
  offsetY: -1,            // Vertical offset from anchor

  // Percentage-based positioning (alternative to anchor)
  row: "25%",             // Vertical position (0%=top, 100%=bottom)
  col: "50%",             // Horizontal position (0%=left, 100%=right)

  // Absolute positioning (overrides anchor/percent)
  row: 5,                 // Exact row position
  col: 10,                // Exact column position

  // Margin from terminal edges
  margin: 2,              // All sides
  margin: { top: 1, right: 2, bottom: 1, left: 2 },

  // Responsive visibility
  visible: (termWidth, termHeight) => termWidth >= 100  // Hide on narrow terminals

  // Focus behavior
  nonCapturing: true       // Don't auto-focus when shown
});

// OverlayHandle methods
handle.hide();              // Permanently remove the overlay
handle.setHidden(true);     // Temporarily hide (can show again)
handle.setHidden(false);    // Show again after hiding
handle.isHidden();          // Check if temporarily hidden
handle.focus();             // Focus and bring to visual front
handle.unfocus();           // Release focus to normal fallback
handle.unfocus({ target: baseComponent }); // Release this overlay to a specific component
handle.unfocus({ target: null });   // Release this overlay and leave focus empty
handle.isFocused();         // Check if overlay has focus
handle.getBounds();         // Get last rendered terminal-relative bounds

handle.unfocus();
// Overlay loses focus; TUI falls back to another visible capturing overlay or the previous focus target.

handle.unfocus({ target: null });
// Overlay loses focus; no component receives input until focus is set again.

// A focused visible overlay reclaims keyboard input after temporary replacement UI
// releases focus. If you want a specific component to receive input while overlays remain
// visible, call handle.unfocus({ target: component }).

// Hide topmost overlay
tui.hideOverlay();

// Check if any visible overlay is active
tui.hasOverlay();
```

**anchor 可选值**：`'center'`、`'top-left'`、`'top-right'`、`'bottom-left'`、`'bottom-right'`、`'top-center'`、`'bottom-center'`、`'left-center'`、`'right-center'`

**解析顺序**：
1. 计算宽度后，应用 `minWidth` 作为宽度下限
2. 位置优先级：绝对 `row`/`col` > 百分比 `row`/`col` > `anchor`
3. `margin` 将最终位置限制在终端边界以内
4. `visible` 回调控制是否渲染覆盖层（每帧调用）

<a id="component-interface"></a>
### Component 接口

所有组件都实现以下接口：

```typescript
interface Component {
  render(width: number): string[];
  handleInput?(data: string): void;
  handleMouse?(event: TuiMouseEvent): TuiMouseEventResult | undefined;
  invalidate(): void;
}
```

| 方法 | 说明 |
|--------|-------------|
| `render(width)` | 返回字符串数组，每个字符串对应一行。每行**不得超过 `width`**，否则 TUI 会报错。使用 `truncateToWidth()` 或手动换行来确保这一点。 |
| `handleInput?(data)` | 当组件拥有焦点并接收到键盘输入时调用。`data` 字符串包含原始终端输入（可能包含 ANSI 转义序列）。 |
| `handleMouse?(event)` | 由 `TuiAltScreen` 调用，用于处理以该组件为目标、经过标准化的指针输入。 |
| `invalidate()` | 必须实现。清除所有缓存的渲染状态，使下一次 `render()` 从头开始。没有缓存渲染状态的组件可以使用空实现。 |

TUI 会在每个渲染行的末尾追加完整的 SGR 重置和 OSC 8 重置。样式不会跨行延续。如果输出带样式的多行文本，应逐行重新应用样式，或使用 `wrapTextWithAnsi()`，以便在换行后的每一行中保留样式。

<a id="mouse-input"></a>
### 鼠标输入

`TuiAltScreen` 会标准化 SGR 鼠标输入，并对组件和覆盖层进行命中测试。事件包含组件局部坐标 `x`/`y`、绝对坐标 `screenX`/`screenY`、边界、按键、修饰键、点击次数和滚轮增量。`TuiMainScreen` 不捕获鼠标输入，因为其回滚历史由终端管理。

```typescript
import type { TuiMouseEvent, TuiMouseEventResult } from "@earendil-works/pi-tui";

handleMouse(event: TuiMouseEvent): TuiMouseEventResult | undefined {
  if (event.type === "click" && event.button === "left") {
    this.expanded = !this.expanded;
    return { handled: true };
  }
  if (event.type === "press" && event.button === "left") {
    return { handled: true, capture: true, focus: true };
  }
  if (event.type === "drag") {
    this.updateFromPointer(event.x, event.y);
    return { handled: true, render: true };
  }
  return undefined;
}
```

返回 `handled` 会抑制渲染器级别的回退行为。`capture` 使后续的 `drag` 和 `release` 事件继续路由到同一个组件。`focus` 请求键盘焦点。可选的 `render` 标志控制重绘：`press`、`click`、`drag` 和 `wheel` 默认触发渲染；`move` 和 `release` 默认不触发。对于发生可见变化的悬停状态，设置 `render: true`；对于已处理但不执行操作的情况，设置 `render: false`。渲染请求会合并，终端输出仍然采用差分渲染。

未处理的手势保留备用屏幕缓冲区模式下的默认行为：滚轮输入滚动最近的 `ScrollView`，并沿链传递未消耗的增量；鼠标主键拖动选择文本；OSC 8 链接会先于父级点击处理器打开；未处理的右键点击保留已配置的粘贴行为。只有按下和释放完整发生且没有拖动时，才会发出 `click` 事件。

使用 `MouseRegion` 可以在不改变组件渲染方式的情况下添加鼠标行为：

```typescript
const collapsible = new MouseRegion(content, (event) => {
  if (event.type !== "click" || event.button !== "left") return undefined;
  expanded = !expanded;
  return { handled: true };
});
```

`Container` 和 `Box` 使用上次渲染帧记录的几何信息，将事件路由到嵌套子组件，因此不会仅为进行命中测试，就因指针移动而重新渲染子组件。显式的 `VStack`、`HStack` 和 `ScrollView` 布局直接使用备用屏幕缓冲区的布局帧。

<a id="focusable-interface-ime-support"></a>
### Focusable 接口（IME 支持）

显示文本光标并需要 IME（Input Method Editor，输入法编辑器）支持的组件，应实现 `Focusable` 接口：

```typescript
import { CURSOR_MARKER, type Component, type Focusable, renderFakeCursor } from "@earendil-works/pi-tui";

class MyInput implements Component, Focusable {
  focused: boolean = false;  // Set by TUI when focus changes
  
  render(width: number): string[] {
    const marker = this.focused ? CURSOR_MARKER : "";
    // Emit marker right before the fake cursor
    return [`> ${beforeCursor}${marker}${renderFakeCursor(atCursor)}${afterCursor}`];
  }

  invalidate(): void {}
}
```

当 `Focusable` 组件拥有焦点时，TUI 会：
1. 在组件上设置 `focused = true`
2. 在渲染输出中查找 `CURSOR_MARKER`（一个零宽 APC 转义序列）
3. 将终端硬件光标定位到该位置
4. 仅在启用 `showHardwareCursor` 时显示硬件光标

`renderFakeCursor()` 用零宽标记包裹光标单元格。TUI 会将其渲染为反色显示。启用 `showHardwareCursor` 时，TUI 会移除紧跟在 `CURSOR_MARKER` 之后的模拟光标，因此获得焦点的组件只显示终端光标。前面没有 `CURSOR_MARKER` 的模拟光标仍会显示。

硬件光标默认保持隐藏。这既保留了模拟光标的渲染，又能为那些在光标隐藏时仍根据其位置定位 IME 候选窗口的终端设置硬件光标位置。某些终端需要可见的硬件光标才能定位 IME；可通过渲染器构造函数的 `showHardwareCursor` 参数或 `setShowHardwareCursor(true)` 启用它。内置的 `Editor` 和 `Input` 组件已经实现了此接口。

**含有嵌入式输入组件的容器组件：** 当容器组件（对话框、选择器等）包含 `Input` 或 `Editor` 子组件时，容器必须实现 `Focusable`，并将焦点状态传递给子组件：

```typescript
import { Container, type Focusable, Input } from "@earendil-works/pi-tui";

class SearchDialog extends Container implements Focusable {
  private searchInput: Input;

  // Propagate focus to child input for IME cursor positioning
  private _focused = false;
  get focused(): boolean { return this._focused; }
  set focused(value: boolean) {
    this._focused = value;
    this.searchInput.focused = value;
  }

  constructor() {
    super();
    this.searchInput = new Input();
    this.addChild(this.searchInput);
  }
}
```

如果不进行这种传递，使用 IME 输入中文、日文、韩文等文字时，候选窗口会显示在错误的位置。

<a id="built-in-components"></a>
## 内置组件

<a id="container"></a>
### Container

将子组件组合在一起。

```typescript
const container = new Container();
container.addChild(component);
container.removeChild(component);
```

<a id="box"></a>
### Box

为所有子组件应用内边距和背景颜色的容器。

```typescript
const box = new Box(
  1,                              // paddingX (default: 1)
  1,                              // paddingY (default: 1)
  (text) => chalk.bgGray(text)   // optional background function
);
box.addChild(new Text("Content"));
box.setBgFn((text) => chalk.bgBlue(text));  // Change background dynamically
```

<a id="text"></a>
### Text

显示多行文本，支持按单词换行和内边距。

```typescript
const text = new Text(
  "Hello World",                  // text content
  1,                              // paddingX (default: 1)
  1,                              // paddingY (default: 1)
  (text) => chalk.bgGray(text)   // optional background function
);
text.setText("Updated text");
text.setCustomBgFn((text) => chalk.bgBlue(text));
```

<a id="truncatedtext"></a>
### TruncatedText

截断以适应视口宽度的单行文本。适用于状态行和标题。

```typescript
const truncated = new TruncatedText(
  "This is a very long line that will be truncated...",
  0,  // paddingX (default: 0)
  0   // paddingY (default: 0)
);
```

<a id="input"></a>
### Input

支持水平滚动的单行文本输入组件。

```typescript
const input = new Input();
input.onSubmit = (value) => console.log(value);
input.setValue("initial");
input.getValue();
```

在备用屏幕缓冲区模式下，点击可定位光标，并让输入组件获得键盘焦点。

**快捷键：**
- `Enter` - 提交
- `Ctrl+A` / `Ctrl+E` - 行首/行尾
- `Ctrl+W` 或 `Alt+Backspace` - 向后删除单词
- `Ctrl+U` - 删除至行首
- `Ctrl+K` - 删除至行尾
- `Ctrl+Left` / `Ctrl+Right` - 按单词导航
- `Alt+Left` / `Alt+Right` - 按单词导航
- 方向键、Backspace、Delete 按预期工作

<a id="editor"></a>
### Editor

多行文本编辑器，支持自动补全、文件补全、粘贴处理，并在内容超出终端高度时进行垂直滚动。

```typescript
interface EditorTheme {
  borderColor: (str: string) => string;
  selectList: SelectListTheme;
}

interface EditorOptions {
  paddingX?: number;  // Horizontal padding (default: 0)
}

const editor = new Editor(tui, theme, options?);  // tui is required for height-aware scrolling
editor.onSubmit = (text) => console.log(text);
editor.onChange = (text) => console.log("Changed:", text);
editor.disableSubmit = true; // Disable submit temporarily
editor.setAutocompleteProvider(provider);
editor.borderColor = (s) => chalk.blue(s); // Change border dynamically
editor.setPaddingX(1); // Update horizontal padding dynamically
editor.getPaddingX();  // Get current padding
```

**特性：**
- 在备用屏幕缓冲区模式下，支持点击定位光标，以及可点击的自动补全行
- 支持按单词换行的多行编辑
- 斜杠命令自动补全（输入 `/`）
- 文件路径自动补全（按 `Tab`）
- 大段粘贴处理（超过 10 行会创建 `[paste #1 +50 lines]` 标记）
- 编辑器上方/下方的水平线
- 模拟光标渲染（隐藏真实光标）

**快捷键：**
- `Enter` - 提交
- `Shift+Enter`、`Ctrl+Enter` 或 `Alt+Enter` - 换行（取决于终端，Alt+Enter 最可靠）
- `Tab` - 自动补全
- `Ctrl+K` - 删除至行尾
- `Ctrl+U` - 删除至行首
- `Ctrl+W` 或 `Alt+Backspace` - 向后删除单词
- `Alt+D` 或 `Alt+Delete` - 向前删除单词
- `Ctrl+A` / `Ctrl+E` - 行首/行尾
- `Ctrl+]` - 向前跳转到指定字符（等待下一次按键，然后将光标移至该字符首次出现的位置）
- `Ctrl+Alt+]` - 向后跳转到指定字符
- 方向键、Backspace、Delete 按预期工作

<a id="markdown"></a>
### Markdown

渲染 Markdown，支持语法高亮和主题。

```typescript
interface MarkdownTheme {
  heading: (text: string) => string;
  link: (text: string) => string;
  linkUrl: (text: string) => string;
  code: (text: string) => string;
  codeBlock: (text: string) => string;
  codeBlockBorder: (text: string) => string;
  quote: (text: string) => string;
  quoteBorder: (text: string) => string;
  hr: (text: string) => string;
  listBullet: (text: string) => string;
  bold: (text: string) => string;
  italic: (text: string) => string;
  strikethrough: (text: string) => string;
  underline: (text: string) => string;
  highlightCode?: (code: string, lang?: string) => string[];
}

interface DefaultTextStyle {
  color?: (text: string) => string;
  bgColor?: (text: string) => string;
  bold?: boolean;
  italic?: boolean;
  strikethrough?: boolean;
  underline?: boolean;
}

const md = new Markdown(
  "# Hello\n\nSome **bold** text",
  1,              // paddingX
  1,              // paddingY
  theme,          // MarkdownTheme
  defaultStyle    // optional DefaultTextStyle
);
md.setText("Updated markdown");
```

**特性：**
- 标题、粗体、斜体、代码块、列表、链接、块引用
- HTML 标签渲染为纯文本
- 通过 `highlightCode` 提供可选的语法高亮
- 支持内边距
- 使用渲染缓存提升性能

<a id="loader"></a>
### Loader

动态加载指示器。

```typescript
const loader = new Loader(
  tui,                              // TUI instance for render updates
  (s) => chalk.cyan(s),            // spinner color function
  (s) => chalk.gray(s),            // message color function
  "Loading..."                      // message (default: "Loading...")
);
loader.start();
loader.setMessage("Still loading...");
loader.stop();
```

<a id="cancellableloader"></a>
### CancellableLoader

扩展 Loader，增加 Escape 键处理，以及用于取消异步操作的 AbortSignal。

```typescript
const loader = new CancellableLoader(
  tui,                              // TUI instance for render updates
  (s) => chalk.cyan(s),            // spinner color function
  (s) => chalk.gray(s),            // message color function
  "Working..."                      // message
);
loader.onAbort = () => done(null); // Called when user presses Escape
doAsyncWork(loader.signal).then(done);
```

**属性：**
- `signal: AbortSignal` - 用户按下 Escape 时中止
- `aborted: boolean` - 加载指示器是否已中止
- `onAbort?: () => void` - 用户按下 Escape 时调用的回调

<a id="selectlist"></a>
### SelectList

支持键盘导航的交互式选择列表。

```typescript
interface SelectItem {
  value: string;
  label: string;
  description?: string;
}

interface SelectListTheme {
  selectedPrefix: (text: string) => string;
  selectedText: (text: string) => string;
  description: (text: string) => string;
  scrollInfo: (text: string) => string;
  noMatch: (text: string) => string;
}

const list = new SelectList(
  [
    { value: "opt1", label: "Option 1", description: "First option" },
    { value: "opt2", label: "Option 2", description: "Second option" },
  ],
  5,      // maxVisible
  theme   // SelectListTheme
);

list.onSelect = (item) => console.log("Selected:", item);
list.onCancel = () => console.log("Cancelled");
list.onSelectionChange = (item) => console.log("Highlighted:", item);
list.setFilter("opt"); // Filter items
```

**操作：**
- 鼠标移动/滚轮：在备用屏幕缓冲区模式下高亮列表行
- 点击：选择一行
- 方向键：导航
- Enter：选择
- Escape：取消

<a id="settingslist"></a>
### SettingsList

支持循环切换值和子菜单的设置面板。

```typescript
interface SettingItem {
  id: string;
  label: string;
  description?: string;
  currentValue: string;
  values?: string[];  // If provided, Enter/Space cycles through these
  submenu?: (currentValue: string, done: (selectedValue?: string) => void) => Component;
}

interface SettingsListTheme {
  label: (text: string, selected: boolean) => string;
  value: (text: string, selected: boolean) => string;
  description: (text: string) => string;
  cursor: string;
  hint: (text: string) => string;
}

const settings = new SettingsList(
  [
    { id: "theme", label: "Theme", currentValue: "dark", values: ["dark", "light"] },
    { id: "model", label: "Model", currentValue: "gpt-4", submenu: (val, done) => modelSelector },
  ],
  10,      // maxVisible
  theme,   // SettingsListTheme
  (id, newValue) => console.log(`${id} changed to ${newValue}`),
  () => console.log("Cancelled")
);
settings.updateValue("theme", "light");
```

**操作：**
- 鼠标移动/滚轮：在备用屏幕缓冲区模式下高亮列表行
- 点击：激活一行
- 方向键：导航
- Enter/Space：激活（循环切换值或打开子菜单）
- Escape：取消

<a id="spacer"></a>
### Spacer

用于垂直间隔的空行。

```typescript
const spacer = new Spacer(2); // 2 empty lines (default: 1)
```

<a id="image"></a>
### Image

在支持 Kitty 图形协议（Kitty、Ghostty、WezTerm）或 iTerm2 内联图片的终端中内联渲染图片。在不支持的终端中回退为文本占位符。

```typescript
interface ImageTheme {
  fallbackColor: (str: string) => string;
}

interface ImageOptions {
  maxWidthCells?: number;
  maxHeightCells?: number;
  filename?: string;
}

const image = new Image(
  base64Data,       // base64-encoded image data
  "image/png",      // MIME type
  theme,            // ImageTheme
  options           // optional ImageOptions
);
tui.addChild(image);
```

支持的格式：PNG、JPEG、GIF、WebP。尺寸会自动从图片头部解析。

<a id="alternate-screen-image-compatibility"></a>
#### 备用屏幕缓冲区的图片兼容性

`TuiAltScreen` 在实现了 Kitty 图形协议的终端（包括 Kitty 和 Ghostty）中支持内联图片和视口部分裁剪。iTerm2 的内联图片协议不提供删除现有图片放置实例，或在滚动时裁剪其源图片的操作。为防止旧图片残留在重绘后的内容之上，`TuiAltScreen` 在 iTerm2 中会将图片组件渲染为文本占位符。`TuiMainScreen` 仍会正常渲染 iTerm2 内联图片。

<a id="autocomplete"></a>
## 自动补全

<a id="combinedautocompleteprovider"></a>
### CombinedAutocompleteProvider

同时支持斜杠命令和文件路径。

```typescript
import { CombinedAutocompleteProvider } from "@earendil-works/pi-tui";

const provider = new CombinedAutocompleteProvider(
  [
    { name: "help", description: "Show help" },
    { name: "clear", description: "Clear screen" },
    { name: "delete", description: "Delete last message" },
  ],
  process.cwd() // base path for file completion
);

editor.setAutocompleteProvider(provider);
```

**特性：**
- 输入 `/` 查看斜杠命令
- 按 `Tab` 补全文件路径
- 支持 `~/`、`./`、`../` 和 `@` 前缀
- 使用 `@` 前缀时，仅筛选可作为附件的文件

<a id="key-detection"></a>
## 按键检测

结合使用 `matchesKey()` 和 `Key` 辅助工具来检测键盘输入（支持 Kitty 键盘协议）：

```typescript
import { matchesKey, Key } from "@earendil-works/pi-tui";

if (matchesKey(data, Key.ctrl("c"))) {
  process.exit(0);
}

if (matchesKey(data, Key.enter)) {
  submit();
} else if (matchesKey(data, Key.escape)) {
  cancel();
} else if (matchesKey(data, Key.up)) {
  moveUp();
}
```

**按键标识符**（使用 `Key.*` 可获得自动补全，也可以使用字符串字面量）：
- 基本按键：`Key.enter`、`Key.escape`、`Key.tab`、`Key.space`、`Key.backspace`、`Key.delete`、`Key.home`、`Key.end`
- 方向键：`Key.up`、`Key.down`、`Key.left`、`Key.right`
- 带修饰键：`Key.ctrl("c")`、`Key.shift("tab")`、`Key.alt("left")`、`Key.ctrlShift("p")`
- 也支持字符串格式：`"enter"`、`"ctrl+c"`、`"shift+tab"`、`"ctrl+shift+p"`

<a id="rendering-modes"></a>
## 渲染模式

`TuiMainScreen` 使用三种渲染策略：

1. **首次渲染**：输出所有行，不清除回滚历史
2. **宽度改变或视口上方发生变化**：清屏并完整重新渲染
3. **常规更新**：将光标移至第一个发生变化的行，清除至末尾，并渲染发生变化的行

`TuiAltScreen` 管理一个终端高度的视口。未显式设置布局根组件时，它保留原有的单文档滚动行为。通过 `setLayoutRoot()`，`VStack`、`HStack` 和嵌套的 `ScrollView` 组件可以预留固定区域，并在受约束的区域中独立滚动。它会原地更新发生变化的视口行，位于底部时跟随流式输出，并在内容增长时保留手动选定的滚动位置。鼠标滚轮和可配置的键盘导航在不修改终端回滚历史的情况下进行滚动，包括在 OSC 133 语义提示符标记之间跳转。滚动条支持悬停展开、拖动滑块和点击轨道跳转。点击 OSC 8 超链接会使用配置的 URL 处理器打开链接。使用鼠标主键拖动可以选择文本，并且除非 `TuiAltScreenOptions.copyOnSelect` 为 `false`，否则会使用 OSC 52 将文本复制到剪贴板；拖动时停留在滚动视图的顶部或底部边缘，会自动滚动并将选区扩展到屏幕外的内容。Kitty 图片支持垂直视口裁剪；iTerm2 内联图片则回退为文本，因为 iTerm2 协议无法在视口重绘期间删除或裁剪图片放置实例。

两个渲染器都会将更新包裹在**同步输出**序列（`\x1b[?2026h` ... `\x1b[?2026l`）中，实现原子化、无闪烁的渲染。

<a id="terminal-interface"></a>
## Terminal 接口

TUI 可以使用任何实现了 `Terminal` 接口的对象：

```typescript
interface Terminal {
  start(onInput: (data: string) => void, onResize: () => void): void;
  stop(): void;
  write(data: string): void;
  get columns(): number;
  get rows(): number;
  moveBy(lines: number): void;
  hideCursor(): void;
  showCursor(): void;
  clearLine(): void;
  clearFromCursor(): void;
  clearScreen(): void;
}
```

**内置实现：**
- `ProcessTerminal` - 使用 `process.stdin/stdout`
- `VirtualTerminal` - 用于测试（使用 `@xterm/headless`）

<a id="utilities"></a>
## 工具函数

```typescript
import { visibleWidth, truncateToWidth, wrapTextWithAnsi } from "@earendil-works/pi-tui";

// Get visible width of string (ignoring ANSI codes)
const width = visibleWidth("\x1b[31mHello\x1b[0m"); // 5

// Truncate string to width (preserving ANSI codes, adds ellipsis)
const truncated = truncateToWidth("Hello World", 8); // "Hello..."

// Truncate without ellipsis
const truncatedNoEllipsis = truncateToWidth("Hello World", 8, ""); // "Hello Wo"

// Wrap text to width (preserving ANSI codes across line breaks)
const lines = wrapTextWithAnsi("This is a long line that needs wrapping", 20);
// ["This is a long line", "that needs wrapping"]
```

<a id="creating-custom-components"></a>
## 创建自定义组件

创建自定义组件时，**`render()` 返回的每一行都不得超过 `width` 参数指定的宽度**。只要有一行比终端更宽，TUI 就会报错。

<a id="handling-input"></a>
### 处理输入

结合使用 `matchesKey()` 和 `Key` 辅助工具来处理键盘输入：

```typescript
import { matchesKey, Key, truncateToWidth } from "@earendil-works/pi-tui";
import type { Component } from "@earendil-works/pi-tui";

class MyInteractiveComponent implements Component {
  private selectedIndex = 0;
  private items = ["Option 1", "Option 2", "Option 3"];
  
  public onSelect?: (index: number) => void;
  public onCancel?: () => void;

  handleInput(data: string): void {
    if (matchesKey(data, Key.up)) {
      this.selectedIndex = Math.max(0, this.selectedIndex - 1);
    } else if (matchesKey(data, Key.down)) {
      this.selectedIndex = Math.min(this.items.length - 1, this.selectedIndex + 1);
    } else if (matchesKey(data, Key.enter)) {
      this.onSelect?.(this.selectedIndex);
    } else if (matchesKey(data, Key.escape) || matchesKey(data, Key.ctrl("c"))) {
      this.onCancel?.();
    }
  }

  render(width: number): string[] {
    return this.items.map((item, i) => {
      const prefix = i === this.selectedIndex ? "> " : "  ";
      return truncateToWidth(prefix + item, width);
    });
  }

  invalidate(): void {}
}
```

<a id="handling-line-width"></a>
### 处理行宽

使用提供的工具函数来确保每行宽度符合要求：

```typescript
import { visibleWidth, truncateToWidth } from "@earendil-works/pi-tui";
import type { Component } from "@earendil-works/pi-tui";

class MyComponent implements Component {
  private text: string;

  constructor(text: string) {
    this.text = text;
  }

  render(width: number): string[] {
    // Option 1: Truncate long lines
    return [truncateToWidth(this.text, width)];

    // Option 2: Check and pad to exact width
    const line = this.text;
    const visible = visibleWidth(line);
    if (visible > width) {
      return [truncateToWidth(line, width)];
    }
    // Pad to exact width (optional, for backgrounds)
    return [line + " ".repeat(width - visible)];
  }

  invalidate(): void {}
}
```

<a id="ansi-code-considerations"></a>
### ANSI 转义码注意事项

`visibleWidth()` 和 `truncateToWidth()` 都能正确处理 ANSI 转义码：

- `visibleWidth()` 在计算宽度时会忽略 ANSI 转义码
- `truncateToWidth()` 在截断时会保留 ANSI 转义码，并正确结束其作用

```typescript
import chalk from "chalk";

const styled = chalk.red("Hello") + " " + chalk.blue("World");
const width = visibleWidth(styled); // 11 (not counting ANSI codes)
const truncated = truncateToWidth(styled, 8); // Red "Hello" + " W..." with proper reset
```

<a id="caching"></a>
### 缓存

为提升性能，组件应缓存其渲染输出，并且只在必要时重新渲染：

```typescript
class CachedComponent implements Component {
  private text: string;
  private cachedWidth?: number;
  private cachedLines?: string[];

  render(width: number): string[] {
    if (this.cachedLines && this.cachedWidth === width) {
      return this.cachedLines;
    }

    const lines = [truncateToWidth(this.text, width)];

    this.cachedWidth = width;
    this.cachedLines = lines;
    return lines;
  }

  invalidate(): void {
    this.cachedWidth = undefined;
    this.cachedLines = undefined;
  }
}
```

<a id="example"></a>
## 示例

完整的聊天界面示例见 `test/chat-simple.ts`，其中包含：
- 使用自定义背景颜色的 Markdown 消息
- 响应期间显示的加载指示器
- 支持自动补全和斜杠命令的编辑器
- 消息之间的间隔

运行示例：
```bash
node test/chat-simple.ts
```

<a id="development"></a>
## 开发

```bash
# Install dependencies (from monorepo root)
npm install

# Run type checking
npm run check

# Run the demo
node test/chat-simple.ts
```

<a id="debug-logging"></a>
### 调试日志

设置 `PI_TUI_WRITE_LOG`，捕获写入 stdout 的原始 ANSI 流。

```bash
PI_TUI_WRITE_LOG=/tmp/tui-ansi.log node test/chat-simple.ts
```
