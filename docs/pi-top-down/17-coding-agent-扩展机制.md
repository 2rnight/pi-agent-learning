# 17 扩展机制

扩展是 pi 的核心差异化设计：**不用 fork 就能改变 agent 的行为。** 官方文档第一句就是它的定位：

> Extensions are TypeScript modules that add executable behavior to Pi. Use one when a workflow needs tools, commands, event handlers, model providers, session state, or terminal UI rather than instructions alone.

## 1. 一个扩展长什么样

```ts
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";

export default function (pi: ExtensionAPI) {
  pi.registerCommand("hello", {
    description: "Show a greeting",
    handler: async (name, ctx) => {
      ctx.ui.notify(`Hello, ${name || "world"}!`, "info");
    },
  });
}
```

- **默认导出工厂函数**，同步或异步都可以（异步会被 await，允许启动时拉配置）；
- 接收 `ExtensionAPI`，在工厂里**注册能力**；
- 用 `jiti` 加载，**TS 不需要预编译**。

加载位置：

```
~/.pi/agent/extensions/          用户级
<repo>/.pi/extensions/           项目级（需项目信任）
pi --extension ./hello.ts        单次（开发用）
pi -e npm:@example/pi-tools      单次（包）
Pi packages（npm/git）           见第 18 章
```

## 2. `ExtensionAPI` 能注册什么

| 方法类别 | 作用 |
|---|---|
| `registerTool(tool)` | 加一个模型可调用的工具（带渲染器） |
| `registerCommand(name, opts)` | 加斜杠命令 |
| `registerShortcut(key, opts)` | 加键绑定 |
| `registerProvider(...)` / provider config | 加模型供应商或覆盖现有 provider |
| `registerVirtualModel(...)` | 加虚拟模型（请求路由） |
| `registerFlag(...)` | 加 CLI 参数 |
| `registerMcpServer` 相关 | 动态加 MCP 服务器 |
| `on(event, handler)` | 订阅 30+ 种事件 |

内置扩展（`src/extensions/index.ts`）就是四个这样写出来的模块：

```ts
export const builtInExtensions: InlineExtension[] = [
  { name: "llama.cpp", factory: llamaExtension, builtin: true },
  // Replaceable: 注册了同名能力的第三方扩展会接管，而不是并行运行
  { name: "codemode",   factory: codemodeExtension,  replaceable: true, builtin: true },
  { name: "tool-search",factory: toolSearchExtension, replaceable: true, builtin: true },
  { name: "mcp",        factory: mcpExtension,       replaceable: true, builtin: true },
];
```

**`replaceable: true` 是个精细的设计**：内置 MCP 扩展可以被第三方同名扩展替换（不共存），避免「两个 MCP 扩展抢同一个工具名」。

## 3. 事件：pi 的可观测面

`ExtensionEvent` 是一个 30+ 成员的联合类型，按生命周期分组：

```
启动/信任    project_trust, resources_discover
会话        session_start, session_info_changed,
            session_before_switch / fork / compact / tree,
            session_compact, session_compact_failed, session_shutdown,
            session_tree, mcp_servers_change
上下文      context, context_with_system
模型        before_provider_request, before_provider_headers,
            after_provider_response, provider_stream_event,
            cache_warming_decision
Agent       before_agent_start, agent_start, agent_end,
            agent_before_settle, agent_settled
轮次        turn_start, turn_end
消息        message_start, message_update, message_end
工具        tool_call, tool_result,
            tool_execution_start, tool_execution_update, tool_execution_end
交互        user_bash, input, model_select, thinking_level_select,
            ui_prompt_start, ui_prompt_end
```

有几组值得单独解释：

### `before_*` 是同步决策点

`session_before_switch` / `session_before_fork` / `session_before_compact` / `session_before_tree` 都返回 `...Result`（例如 `SessionBeforeCompactResult`）。它们可以对操作做**否决/修改**，而 `session_*`（无 before）只是通知。

### `context` / `context_with_system`

这正是第 10 章 `transformContext` 的扩展入口：扩展可以裁剪或注入消息。`context_with_system` 版本额外能改系统提示词（`ContextWithSystemEvent`）。

### `agent_before_settle` vs `agent_settled`

文档解释得很清楚：

> `agent_before_settle` is the final actionable boundary: it can append entries and request one continuation.
> `agent_settled` is final and notification-only.

也就是说：`before_settle` 是「最后一次机会再推动一轮」（可以返回继续），`settled` 只是「真的结束了」。

### `tool_call` 可以被拦截并改写参数

`ToolCallEvent`（按工具细分：`ReadToolCallEvent`、`BashToolCallEvent`、`EditToolCallEvent`…）+ `ToolCallEventResult`。这是**权限控制/路径保护**的实现位置：

```ts
pi.on("tool_call", async (event, ctx) => {
  if (event.type === "tool_call" && isDangerous(event)) return { block: true, reason: "..." };
});
```

注意：这和 agent-core 的 `beforeToolCall` 不是一回事。`beforeToolCall` 是运行时钩子（任何用 agent-core 的程序都能用），扩展事件是 coding-agent 这一层的（带 ctx、带 UI、可被其他扩展观察）。

## 4. 加载器：`jiti` + 虚拟模块

`core/extensions/loader.ts`：

```ts
const isNodeSeaBinary = ("sea" in process.features && process.features.sea === true) || ...;
const isTypeScriptSourceRuntime = !isBunBinary && path.extname(fileURLToPath(import.meta.url)) === ".ts";
const usesEmbeddedModules = isBunBinary || isNodeSeaBinary || isBundledNode;

function getCreateJiti() {
  createJitiPromise ??= (usesEmbeddedModules ? import("./jiti-static-loader.ts") : import("./jiti-loader.ts"))...
}
```

这段代码体现了 pi 要同时支持**四种发布形态**：

| 形态 | 判定 |
|---|---|
| Bun 编译二进制 | `import.meta.url` 含 `$bunfs`/`~BUN` |
| Node SEA（single executable app） | `process.features.sea` |
| esbuild 打包的 Node 分发 | `PI_BUNDLED_NODE` |
| 源码/普通 npm 安装 | 都不是 |

每种形态下「怎么把用户写的 TS 加载进来」都不同，所以要两套 jiti 配置。**这是「一个 CLI 要在 4 种运行时发出去」的真实代价。**

`virtual-modules.ts` 还提供了虚拟模块（让扩展能 import 一些由 pi 注入的东西，而不需要真的依赖）。

## 5. 运行时生命周期（文档里的硬规则）

扩展作者最容易踩的坑，文档都写了：

> Do not start processes, sockets, watchers, or timers in the factory because some invocations load extensions without starting a session.
> Start long-lived resources from `session_start` or from the command or tool that needs them.
> Close session-scoped resources from an idempotent `session_shutdown` handler.

原因：`pi --help`、`pi list`、`pi skills` 这类命令也会加载扩展，但不会开会话。工厂里起一个 MCP 连接会导致「只想看帮助却启动了一堆进程」。

另一条（第 12 章提过）：`ctx.reload()` / `newSession()` 之后旧 ctx 失效。源码里有运行时检查：

> This extension ctx is stale after session replacement or reload. Do not use a captured `pi` or command ctx after `ctx.newSession()`, `ctx.fork()`, `ctx.switchSession()`, or `ctx.reload()`.

## 6. 信任模型：扩展等于任意代码执行

`docs/extensions.md` 第一段就警告：

> An extension runs inside the Pi process with the same operating-system permissions. It can inspect prompts, tool calls, files, credentials, and session history, so load extensions only from sources you trust.

机制上：

- **项目级扩展**（`.pi/extensions/`）需要先通过 `project_trust`；
- `project_trust` 事件本身只能由**个人扩展或显式 `--extension`** 参与（不能用项目扩展决定自己是否被信任——那显然荒谬）；
- 用户级扩展无需信任（你自己装的）；
- `pi -ne`（no extensions）是逃生舱，`main.ts` 里就有提示：

```ts
const EXTENSION_LOAD_FAILURE_HINT = `Hint: Start without extensions using "${APP_NAME} -ne".`;
```

**注意 pi 明确不沙箱扩展**（README/SECURITY 都说没有内置权限系统）。想要边界就容器化（`docs/containerization.md`）。

## 7. 扩展的四种「形态」

| 形态 | 例子 | 特点 |
|---|---|---|
| 单个 `.ts` 文件 | `~/.pi/agent/extensions/hello.ts` | 最简单 |
| 目录 + `index.ts` | 多文件扩展 | 可以有 `package.json` 依赖 |
| Pi package | `pi install npm:@example/pi-tools` | 分发用，含资源目录 |
| 内联扩展 | `builtInExtensions` | pi 自己带的 |

## 8. 这个仓库自己的扩展

`.pi/extensions/` 里有四个：

```
import-repro.ts        # 从会话里重建复现脚本
prompt-url-widget.ts   # 提示词 URL 挂件
redraws.ts             # 重绘调试
tps.ts                 # tokens per second 显示
```

**这些是「用扩展调试 pi 自身」的例子**。`tps.ts`（吞吐量显示）尤其典型：核心不需要这个功能，但开发者想要，于是写成扩展——这正是 README 说的「adapt Pi to your workflows」。

## 9. 本章要点

- 扩展 = 默认导出工厂函数的 TS 模块，用 jiti 免编译加载；能注册工具/命令/快捷键/provider/虚拟模型/事件。
- 30+ 事件覆盖启动、会话、上下文、模型、agent、轮次、消息、工具、交互九组；`before_*` 可否决，`*_settled` 只通知。
- 工厂里不要起长连接/定时器（很多命令只加载扩展不开会话）。
- 扩展以进程权限运行，项目扩展需要 `project_trust`，`-ne` 是逃生舱。
- 内置扩展可被同名第三方扩展 `replaceable` 接管。

下一章：[18 资源、设置与包管理](18-coding-agent-资源与包管理.md)
