# 11 coding-agent 分层架构

`packages/coding-agent` 是产品本体：85k 行、299 个文件。但它内部的分层非常干净，只要认清五个目录的职责就不会迷路。

## 1. 五个目录，五种职责

```
packages/coding-agent/src/
├── main.ts          (998 行)  CLI 入口：解析参数 → 转成 createAgentSession 选项 → 跑模式
├── cli/             (12 文件) 参数解析、认证子命令、文件处理、会话选择、信任决策
├── modes/           (4 种)   interactive / print / json / rpc —— 只负责 I/O
├── core/            (60+ 文件) 产品内核：AgentSession、会话、工具、设置、资源、扩展
├── extensions/      (4 个内置) codemode / mcp / tool-search / llama.cpp
├── bun/             打包成 Bun 单文件二进制时的运行时适配
├── experimental/    实验性的 server/worker/plugin/durable 集成
└── client/          SDK 里的客户端封装
```

**判据**：如果一段代码在描述「怎么显示」→ `modes`；「怎么持久化/配置/扩展」→ `core`；「怎么把命令行变成选项」→ `cli`/`main.ts`。

### `main.ts` 的自我定位

文件头注释：

> This file handles CLI argument parsing and translates them into `createAgentSession()` options. The SDK does the heavy lifting.

它导入的东西暴露了它要处理的现实复杂度：

```ts
import { parseArgs, printHelp } from "./cli/args.ts";
import { checkProviderAuth, getProviderCredential } from "./cli/auth-check.ts";
import { parseAuthCommand, ... } from "./cli/auth-command.ts";
import { createProjectTrustContext } from "./cli/project-trust.ts";
import { selectSession } from "./cli/session-picker.ts";
import { shouldRunFirstTimeSetup, showStartupSelector } from "./cli/startup-ui.ts";
import { createAgentSessionRuntime } from "./core/agent-session-runtime.ts";
```

注意最后一句：**连 main 都不直接 `new AgentSession`**，而是走 `createAgentSessionRuntime()`（runtime 负责在「切换会话/重载扩展」时替换内部 session，而 main 保持稳定引用）。

`main.ts` 里还有一个模式判定：

```ts
function resolveAppMode(parsed: Args, stdinIsTTY: boolean, stdoutIsTTY: boolean): AppMode {
  if (parsed.mode === "rpc") return "rpc";
  if (parsed.mode === "json") return "json";
  if (parsed.print || !stdinIsTTY || !stdoutIsTTY) return "print";
  return "interactive";
}
```

**「不是 TTY 就自动降级成 print 模式」**——这就是为什么 `echo "hi" | pi` 和 `pi > out.txt` 都能工作。

## 2. `core/` 内部再分层

`core/` 有 60+ 文件，但按职责可以分七组：

| 组 | 代表文件 | 职责 |
|---|---|---|
| **会话生命周期** | `agent-session.ts`（4349 行）、`agent-session-runtime.ts`、`agent-session-services.ts`、`sdk.ts` | 把 agent-core 包成产品 |
| **持久化** | `session-manager.ts`（2013 行）、`session-cwd.ts`、`session-export.ts` | JSONL 树、投影、导出 |
| **模型** | `model-runtime.ts`、`model-registry.ts`、`model-resolver.ts`、`models-store.ts`、`virtual-models.ts`、`provider-composer.ts`、`provider-attribution.ts` | 选模型、配额、自定义端点、虚拟模型 |
| **上下文** | `system-prompt.ts`、`messages.ts`、`skills.ts`、`prompt-templates.ts`、`compaction/`（4 文件） | 组装提示词与压缩 |
| **工具** | `tools/`（19 文件） | read/bash/edit/write/grep/find/ls/powershell + 截断/diff/渲染 |
| **扩展** | `extensions/`（loader/runner/types）、`resource-loader.ts`、`package-manager.ts`、`mcp-servers.ts` | 加载与运行扩展、资源发现、包管理 |
| **配置与安全** | `settings-manager.ts`（1533 行）、`auth-storage.ts`、`trust-manager.ts`、`project-trust.ts`、`http-dispatcher.ts` | 设置、凭证、信任、代理 |

规模提示：`agent-session.ts` + `session-manager.ts` + `settings-manager.ts` + `package-manager.ts` 四个文件就占了约 9.9k 行 —— **产品的复杂度主要不在算法，而在状态与配置的组合**。

## 3. `modes/`：四层薄壳

```
modes/
├── interactive/        interactive-mode.ts (7072 行) + components/ (20+) + theme/
├── print-mode.ts       runPrintMode：跑一次，输出最终文本/JSON
├── json-event.ts       JSONL 事件序列化
├── rpc/                rpc-mode.ts (819) + rpc-client.ts + rpc-types.ts
└── index.ts            统一导出
```

`interactive-mode.ts` 7072 行是仓库最大的文件——它要处理：键盘输入、斜杠命令、自动补全、编辑器、消息渲染、工具执行展示、diff、图片、鼠标、主题、easter egg、3D 彩蛋（`easter-egg-3d.ts` 1292 行）。

**为什么不拆**：它是「一个交互循环」的状态机，拆开会让跨模块的状态传递更糟。这是一次刻意的取舍（极端一点的对比是 `tui-plan.md` 里对渲染器重构的思考）。

## 4. 依赖方向（关键）

```
main.ts ──► cli/ ──► core/ ──► (agent-core, ai, tui, mcp, codemode)
  │                    │
  └────► modes/ ───────┘
```

一条硬规则（在 `AGENTS.md` 里也有体现）：**`core/` 不能 import `modes/`**。但有一个现实例外——`core/agent-session.ts` 导入了 `../modes/interactive/theme/theme.ts`。为什么？

因为主题（theme）本质上是「把语义 token 映射成颜色」的**数据**，工具渲染需要它（工具结果在 TUI 里要上色）。主题被放在 `modes/interactive/theme/` 是历史位置，但它其实是跨层的资源。这是一个「架构上不完美但工程上合理」的接缝，值得注意。

## 5. 装配流程（从 CLI 到可运行）

```
main()
 ├─ parseArgs()                        cli/args.ts
 ├─ 提前截获 auth 子命令               cli/auth-command.ts
 ├─ 读 stdin（若非 TTY）               readPipedStdin()
 ├─ ProjectTrustStore 解析信任         core/trust-manager.ts
 ├─ SettingsManager（global + project） core/settings-manager.ts
 ├─ ModelRuntime + ModelRegistry        core/model-runtime.ts
 ├─ AuthStorage                         core/auth-storage.ts
 ├─ ResourceLoader.discover()           core/resource-loader.ts
 │    ├─ extensions（含 package-manager 装的）
 │    ├─ skills
 │    ├─ prompt templates
 │    └─ context files（AGENTS.md …）
 ├─ createAgentSessionRuntime()         core/agent-session-runtime.ts
 │    └─ createAgentSession()           core/sdk.ts
 │         ├─ Agent(agent-core)
 │         ├─ SessionManager
 │         ├─ ExtensionRunner
 │         └─ 工具集合（built-in + 扩展注册的）
 └─ 按 AppMode 分发到 InteractiveMode / runPrintMode / runRpcMode
```

**注意顺序里的一个安全点**：`ProjectTrustStore` 在「加载项目资源」之前解析。这就是 `how-pi-works.md` 说的：

> Pi resolves project trust before loading project settings and resources.

因为项目级的 `extensions/` 会在 pi 进程里执行任意代码——**必须先问用户是否信任这个目录**。

## 6. 一个反例：`experimental/`

`src/experimental/` 里有 `server.ts`、`session-worker.ts`、`session-worker-manager.ts`、`coordinator.ts`、`plugin.ts`、`durable/`、`vacation/`。

它存在说明：**pi 正在把「单进程 CLI」往「服务器 + worker + 远程附着」演进**，但还没决定是否合并进主线。对照第 23–25 章的 chord 体系，你会看到两条殊途同归的路线：

- `experimental/` 走的是「用 chord + protocol + durable 把 coding-agent 服务化」；
- `durable` 包走的是「独立实现一个可恢复的 harness」。

## 7. 本章要点

- 五层职责：`main`（接线）/ `cli`（参数）/ `modes`（I/O）/ `core`（内核）/ `extensions`（内置扩展）。
- `main.ts` 明确「只做选项翻译」，真正的装配在 `core/sdk.ts` 与 `agent-session-runtime.ts`。
- 非 TTY 自动降级到 print 模式；auth 子命令在完整解析前被截获。
- `core/` 的复杂度集中在 `agent-session` / `session-manager` / `settings-manager` / `package-manager`。
- 加载顺序里 **trust 在 resources 之前**，这是安全边界而非实现细节。

下一章：[12 AgentSession](12-coding-agent-AgentSession.md)
