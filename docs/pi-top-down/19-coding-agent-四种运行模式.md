# 19 四种运行模式（以及 SDK）

同一个内核（`AgentSession`），五套外壳。这一章讲它们各自适合什么、内部怎么接线。

## 1. 对照表

| 模式 | 进程边界 | 控制方式 | 适合 |
|---|---|---|---|
| **interactive** | 同进程 | 键盘/鼠标，TUI | 人在终端里干活 |
| **print** | 同进程 | CLI 参数 + stdin | 一次性任务、脚本 |
| **json** | 同进程 | CLI 参数 + stdin | 消费结构化事件 |
| **rpc** | 子进程 | stdin/stdout JSONL | 其他语言、IDE、进程隔离 |
| **SDK** | 同进程 | TS 方法调用 | Node/Bun 宿主应用 |

`modes/index.ts` 的导出直接对应前四个：

```ts
export { InteractiveMode, ... } from "./interactive/interactive-mode.ts";
export { runPrintMode, type PrintModeOptions } from "./print-mode.ts";
export { runRpcMode } from "./rpc/rpc-mode.ts";
export { RpcClient, ... } from "./rpc/rpc-client.ts";
```

## 2. 模式选择：自动降级

`main.ts`：

```ts
function resolveAppMode(parsed: Args, stdinIsTTY: boolean, stdoutIsTTY: boolean): AppMode {
  if (parsed.mode === "rpc") return "rpc";
  if (parsed.mode === "json") return "json";
  if (parsed.print || !stdinIsTTY || !stdoutIsTTY) return "print";
  return "interactive";
}
```

推论：

```bash
pi "解释这个仓库"              # interactive
pi -p "解释这个仓库"           # print
echo "解释这个仓库" | pi       # stdin 不是 TTY → print
pi > out.txt                   # stdout 不是 TTY → print
pi --mode json "..."           # json
pi --mode rpc                  # rpc
```

**这个设计让 pi 天然可脚本化**：不需要额外的 `--non-interactive` 标志，管道重定向自动生效。

## 3. interactive 模式

`InteractiveMode`（`interactive-mode.ts`，7072 行）是最大的文件。它做的事：

- 用 `pi-tui` 构建界面（编辑器、消息列表、工具执行块、footer、overlay）；
- 处理输入：普通文本、`/命令`、`@文件`、`!shell`、粘贴、图片拖入、键绑定；
- 订阅 `AgentSession` 事件 → 更新组件；
- 处理扩展的 UI 请求（对话框、选择器、widget）；
- 主题、自动补全、鼠标、滚动。

**为什么它这么肥**：它是一个「终端里的应用」，不是一段渲染代码。slash 命令面板、会话选择器、树导航器、设置面板、模型选择器……每一个都是独立的交互子系统（`components/` 下 20+ 文件，`tree-selector.ts` 单文件 1435 行）。

### 三个「非 UI」职责被放在这里

1. **输入解析**（`@file`、`!shell`）——因为它依赖编辑器状态；
2. **扩展 UI 桥接**（`ExtensionUIContext` 的实现）——因为只有它有终端；
3. **本地命令**（`/model`、`/compact`、`/resume`…）——因为它们要改 UI 状态。

## 4. print 模式

`runPrintMode()`。语义：跑一次 prompt，输出结果，退出。

```bash
pi -p "把 README 翻成中文"
```

输出形态由 `toPrintOutputMode()` 决定：`"text"` 或 `"json"`。它订阅事件但只关心最终消息，**不渲染中间过程**。

一个细节：`main.ts` 里有 `isPlainRuntimeMetadataCommand()`——`--help`、`--list-models` 这类命令不启动会话，直接输出。

## 5. json 模式

输出 **JSONL 事件流**（`modes/json-event.ts` 定义 `JsonAgentSessionEvent`）。适合：

- 让脚本消费「工具调用/文本增量」等结构；
- 把 pi 的事件转进别的系统（日志、评测、可视化）。

## 6. rpc 模式

`runRpcMode()`（819 行）。协议有**四类记录**（`docs/rpc.md`）：

| 方向 | 记录 | 用途 |
|---|---|---|
| stdin | **Command** | 让 pi 提示、查状态、改配置、管会话 |
| stdout | **response** | 一条命令是否成功 + 数据 |
| stdout | **Session event** | 流式 run/message/tool/queue/compaction/retry 事件 |
| 双向 | **Extension UI record** | 扩展的交互请求/响应 |

命令/响应靠可选 `id` 关联：

```json
{"id":"req-1","type":"get_state"}
```

**为什么需要 RPC 而不是 SDK**：SDK 要求宿主是 Node/Bun 且愿意共享进程。RPC 让「任何语言 + 进程隔离 + 崩溃不拖垮宿主」成为可能。源码里还提供了 `RpcClient`（TS 侧封装：启动进程、关联响应、类型化命令、事件监听）。

RPC 模式的限制（文档明说）：**拒绝 `@file` 参数**，因为那是终端编辑器语义，应该由客户端把内容放进 `prompt` 命令。

## 7. SDK 模式

```ts
import { createAgentSession } from "@earendil-works/pi-coding-agent";

const { session } = await createAgentSession();
try {
  await session.prompt("What files are in the current directory?");
  console.log(session.getLastAssistantText());
} finally {
  session.dispose();
}
```

SDK 的价值在于「完整的可编程控制」：

- 传 `agentDir` / `cwd` / 模型；
- 传入自定义 `ExtensionFactory`（而不是从磁盘加载）；
- 用 `SessionManager` 传入历史（内存或持久）；
- 访问 `session.agent` 的底层 API（第 07 章那张钩子表全部可用）。

文档强调了两个「SDK 与 CLI 一致」的点：

> This uses the working directory, discovered resources, stored settings, and configured credentials.

也就是说 SDK **复用同一套配置与资源发现**，不是你手动拼出来的简化版。

## 8. 五者的共同内核，以及差异点

```
                    ┌──────────────┐
   interactive ────►│              │
   print       ────►│ AgentSession │◄──── rpc
   json        ────►│              │◄──── SDK
                    └──────────────┘
                            ▲
                       Agent (agent-core)
```

差异只在三处：

| 差异 | interactive | print | json | rpc | SDK |
|---|---|---|---|---|---|
| 事件消费 | 渲染组件 | 取最后文本 | 序列化 | 序列化 + 关联命令 | 用户回调 |
| 人机交互 | 完整 | 无 | 无 | 由客户端实现 | 由宿主实现 |
| 扩展 UI | 终端组件 | 不支持/降级 | 同 print | `rpc-extension-ui` 转发 | 宿主提供 |

注意最后一行：**扩展在不同模式下能做的事不同**。扩展能查到当前模式（`ExtensionMode = "tui" | "rpc" | "json" | "print"`），据此决定要不要弹对话框。

## 9. 一个值得注意的横切设计：输出守卫

`core/output-guard.ts` 的 `takeOverStdout()` / `restoreStdout()`。为什么需要？

因为 interactive 模式要独占终端（alt screen / raw mode），但包里同时有「会打印东西」的东西（扩展的 `console.log`、子进程输出、更新提示）。输出守卫把 stdout 接管，避免这些输出打乱渲染。

`package-manager.ts` 里甚至有一句 `isStdoutTakenOver()` 判断——**包管理器知道「别在 TUI 渲染期间乱打印」**。这种细节是「终端应用」与「库」的分水岭。

## 10. 本章要点

- 五种形态共用 `AgentSession`；模式只决定「事件怎么消费、人怎么交互」。
- 非 TTY 自动降级成 print，让 pi 天然可管道化。
- RPC 是四类 JSONL 记录 + id 关联；适合跨语言与隔离。
- SDK 是完整可编程入口，复用同一套配置/资源/凭证。
- 扩展要感知 `ExtensionMode`；输出守卫保证 TUI 不被杂散打印破坏。

下一章：[20 pi-tui 终端渲染](20-pi-tui-终端渲染.md)
