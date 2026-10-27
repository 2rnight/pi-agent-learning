# 20 pi-tui：终端渲染

`pi-tui` 是一个独立可用的终端 UI 库（19k 行），pi 的交互界面建在它上面。它的核心命题只有一个：**在终端里做高频更新，不闪烁。**

## 1. 两个渲染器，两种哲学

```ts
import { type TUI, TuiMainScreen, TuiAltScreen } from "@earendil-works/pi-tui";

const tui: TUI = new TuiMainScreen(terminal);
// 或
const tui: TUI = new TuiAltScreen(terminal);
```

| | `TuiMainScreen` | `TuiAltScreen` |
|---|---|---|
| 缓冲区 | 主屏幕（保留终端 scrollback） | 备用屏幕（fixed viewport） |
| 滚动 | 终端自己管 | 应用自己管（`ScrollView`） |
| 退出 | 内容留在屏幕上 | 恢复主屏，打印完整最终文档 |
| 布局 | 只支持线性追加 | 支持 `VStack`/`HStack`/`ScrollView` 约束布局 |

`isViewportTUI(tui)` 用来判断当前是否支持视口布局：

```ts
export const VIEWPORT_TUI = Symbol.for("@earendil-works/pi-tui/viewport");
export interface ViewportTUI extends TUI {
  readonly [VIEWPORT_TUI]: true;
  setLayoutRoot(component: Component | undefined): void;
}
```

**为什么用 Symbol 而不是 `instanceof`**：避免跨包/多副本时的类身份问题。这是一个「库作者才会想到」的细节。

## 2. 差分渲染（differential rendering）

核心思想：**只重写变化的行，而不是整屏重绘**。

```
上一帧的行数组   previousLines: string[]
这一帧的行数组   nextLines: string[]
             ↓ 比较
   只对不同的行发 ANSI 光标定位 + 内容
```

`tui-main-screen.ts` 里有几个关键实现点：

```ts
// 合成 overlay 之后再比较
// Composite overlays into the rendered lines (before differential compare)

output.append("\x1b[?2026h"); // Begin synchronized output
...
output.append("\x1b[?2026l"); // End synchronized output
```

### 同步输出（CSI 2026）

`\x1b[?2026h` / `\x1b[?2026l` 是「同步更新」私有模式。终端在 begin/end 之间**不刷屏**，等 end 再一次性呈现。效果：**即使你写了 200 行更新，用户看到的是一次原子切换，不会看到中间态闪动**。

支持它的终端（Kitty、WezTerm、iTerm2、Windows Terminal 等）体验极好；不支持的终端会忽略这两个序列（无害降级）。

### 节流

```ts
private static readonly MIN_RENDER_INTERVAL_MS = 16;
```

`TuiBase` 把渲染请求按最多 **~60fps** 合并（`requestRender` → 定时器 → `renderNow`）。流式输出时每个 token 都调用 `requestRender`，但实际最多 60 次/秒——**这是 token 速率与终端写入速率之间的解耦**。

配合 `fullRedraws` 计数器（`readonly fullRedraws: number`）可以观测「有多少次没能走增量路径」——一个很好的自诊断指标（仓库 `.pi/extensions/redraws.ts` 就是干这个的）。

## 3. `TUI` 接口：组件树 + 焦点 + 覆盖层 + 输入

```ts
export interface TUI extends Component {
  readonly mode: TuiMode;
  children: Component[];
  terminal: Terminal;
  addChild(component: Component): void;
  removeChild(component: Component): void;
  setFocus(component: Component | null): void;
  showOverlay(component: Component, options?: OverlayOptions): OverlayHandle;
  hideOverlay(): void;
  start(): void;
  stop(options?: TuiStopOptions): void;
  requestRender(force?: boolean): void;
  renderNow(force?: boolean): void;
  addInputListener(listener: TuiInputListener): () => void;
  queryTerminalColors(options): Promise<TerminalColors>;
}
```

要素不多但够用：

- **组件树**：`Component` 只有一个必需方法 `render()`，返回字符串行数组。这是「终端版 React」的最小实现。
- **焦点**：只有聚焦的组件收到键盘输入。
- **Overlay**：对话框/选择器不需要重建整棵树，叠一层再合成。
- **输入监听**：全局监听器（用于 Ctrl+C 之类）优先于焦点组件。

## 4. 输入处理：终端的坑都在这

`src/` 下与输入相关的文件暗示了复杂度：

```
stdin-buffer.ts        # 分片/合并 stdin 数据（一个键可能分多次到达）
keys.ts                # 按键识别 + matchesKey()
keybindings.ts         # 可配置键绑定
native-modifiers.ts    # 平台特定修饰键
word-navigation.ts     # 按词移动光标
kill-ring.ts           # Emacs 风格 kill ring
undo-stack.ts          # 撤销
```

配合 `AGENTS.md` 的一条规则：

> Never hardcode key checks (e.g. `matchesKey(keyData, "ctrl+x")`). Add defaults to `DEFAULT_EDITOR_KEYBINDINGS` or `DEFAULT_APP_KEYBINDINGS` so they stay configurable.

**「键绑定必须可配置」是硬规定**：终端用户对快捷键有强偏好，硬编码会引发无休止的 issue。

## 5. 编辑器组件

`components/editor.ts` 是仓库里最复杂的 UI 组件之一（`editor-component.ts` 定义接口）。它要实现：

- 多行编辑、光标移动（字符/词/行/文档）；
- 撤销栈、kill ring；
- 粘贴（含 bracketed paste，>10 行时用标记包裹）；
- 自动补全（文件路径 `autocomplete.ts`、斜杠命令）；
- 图片粘贴；
- 与 `tui.setFocus()` 的协作。

**「bracketed paste」为什么重要**：不加这个，粘贴一段带换行的文本会被终端当成「多次回车」，直接提交多次。pi-tui 用 `\x1b[200~` / `\x1b[201~` 包裹，并在 >10 行时特殊处理。

## 6. 内置组件清单

```
Text, TruncatedText, Input, Editor, Markdown, Loader,
SelectList, SettingsList, MouseRegion, Spacer, Image,
Box, Container, VStack, HStack, ScrollView
```

两类值得单独提：

- **`Markdown`**：渲染模型输出（代码高亮 + 表格 + LaTeX，见 `latex.ts`、`highlightCode`）。
- **`Image`**：用 Kitty / iTerm2 图形协议在终端里显示图片。`terminal-image.ts` + `native/`（三个平台的原生模块，负责剪贴板等）—— **这就是为什么 `pi-tui` 要带 C/Objective-C 源码**。

## 7. 颜色：OKLCH / OKHSL 与终端色

`colors.ts` / `oklab.ts` / `terminal-colors.ts`：

```ts
const accent = parseColor("oklch(70% 0.12 220)");
const bg = parseColor("#20242a");
const fg = mixColors(accent, bg, 0.2);
const text = styleText("Ready", { fg, bg, bold: true }, getTerminalColorMode());
```

- `Color` 三态：索引色（0-15 用终端调色板）、sRGB、OKLCH；
- **所有颜色都能转 sRGB**，所以 `mixColors()` 永远可用；
- `styleText()` 按终端能力降级到 truecolor / 256 色；
- 支持 OKHSL（饱和度相对于 sRGB 色域，保证「看起来一样鲜艳」）；
- README 提醒：**转换不缓存**，每帧都用的颜色应该预先转换一次（性能建议）。

主题（`modes/interactive/theme/`）就是「语义 token → Color」的映射表，支持 JSON 主题 + 校验 + 热重载。

## 8. 为什么这些细节值得学

`tui-plan.md`（36KB）是这个库的设计文档。翻一下会发现作者在纠结的问题都很「终端」：

- 主屏 vs 备用屏的取舍（scrollback 属于用户还是应用？）
- 退出时如何「打印完整最终文档」让历史留在 shell 里
- 视口布局如何分配 `basis/grow/shrink/minSize`
- 鼠标滚轮的 `overscroll: "chain"` 如何传递给父容器

**这些是「写终端应用」与「写 Web 应用」真正的差异**，也是 `pi-tui` 作为独立库的价值。

## 9. 本章要点

- 两个渲染器：主屏（保留 scrollback）与备用屏（应用管滚动/布局）。
- 差分渲染 + 同步输出（CSI 2026）+ 16ms 节流 = 高频更新不闪。
- 组件只是 `render(): string[]`；焦点/overlay/输入监听构成应用骨架。
- 终端输入的复杂度集中在 stdin 分片、键绑定可配置、bracketed paste、图片协议。
- 颜色系统支持 OKLCH/OKHSL，可混色、按能力降级。

下一章：[21 pi-mcp 与 pi-codemode](21-pi-mcp与pi-codemode.md)
