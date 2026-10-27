# 21 pi-mcp 与 pi-codemode：接外部世界的两种方式

两个包都在解决「让模型能做更多事」，但路径相反：

| | `pi-mcp` | `pi-codemode` |
|---|---|---|
| 思路 | **协议接入**：把外部工具暴露成工具 | **代码编排**：让模型写代码调用工具 |
| 上下文成本 | 每个工具都进上下文（名字+描述+schema） | 只有一段脚本和最终输出进上下文 |
| 执行地点 | 外部 MCP 服务器进程 | 本地 QuickJS/WASM 沙箱 |
| 依赖 | 零 pi 依赖 | 零 pi 依赖 |

两者都很小（3.2k / 1.7k 行），且都是**独立可用**的——不依赖 `pi-agent-core`。这是「工具生态」与「agent 运行时」解耦的体现。

## 1. pi-mcp：一个不依赖官方 SDK 的 MCP 客户端

> A small, standalone Model Context Protocol client. It does not depend on the official MCP SDK or other pi packages.

```ts
import { McpClient, StdioTransport } from "@earendil-works/pi-mcp";

const transport = new StdioTransport({
  command: "npx",
  args: ["-y", "@modelcontextprotocol/server-filesystem", "/workspace"],
});
const client = new McpClient({
  name: "my-client",
  version: "1.0.0",
  roots: [{ uri: "file:///workspace", name: "workspace" }],
});

await client.connect(transport);
const tools = await client.listTools();
const result = await client.callTool("search", { query: "MCP" });
await client.close();
```

### 目录结构

```
src/
├── index.ts
├── protocol/       # JSON-RPC 2.0 与 MCP 消息类型
├── transports/     # stdio / Streamable HTTP（+ 内存测试传输）
├── oauth/          # MCP 服务器也可能要 OAuth
└── testing/        # 测试用内存传输
```

### 关键工具函数：`toLlmContent(result)`

MCP 的 `CallToolResult` 结构与 pi 的 `TextContent|ImageContent` 不同。这个函数负责转换：

- 文本、图片 → 直接透传；
- 内嵌的 text/image resource → 解包；
- 音频、resource link、二进制 resource → 变成简短文本占位（模型看不了，至少知道有东西）；
- 没有 content 但有 `structuredContent` → 变成它的 JSON。

**「不能表示的降级成文本占位」是 MCP 集成的核心工程问题**——不处理就会出现「模型看不到工具返回了什么」的静默失败。

### 包装成 `AgentTool`

README 给了完整示例，其中有一个细节值得注意：

```ts
name: `mcp_${tool.name}`.replace(/[^A-Za-z0-9_-]/g, "_").slice(0, 64),
```

供应商对工具名的限制是 **最多 64 字符、只允许 `[A-Za-z0-9_-]`**。所以 MCP 工具名必须清洗 + 截断 + 加前缀（避免和内置工具撞名）。

### 在 coding-agent 里的形态

`src/extensions/mcp/` 是内置扩展（可被第三方替换）：

- 配置来自 `<agent-dir>/mcp.json` 与 `.pi/mcp.json`（`core/mcp-servers.ts`）；
- 工具按 `ToolExposure` 决定是否进上下文（可以「藏起来，用 tool_search 找」）；
- `mcp_servers_change` 事件允许运行中增删服务器。

## 2. pi-codemode：把「工具调用」变成「写脚本」

> Runs model-written JavaScript in a QuickJS VM (compiled to WebAssembly) where the only capability is calling injected tools. **Nested tool calls never enter the LLM context; only the script's output and return value do.**

这句话是整个包的价值主张。对比一下两种做法：

**方式 A（直接工具调用）**：模型读 50 个文件 → 50 次 tool call/result 进上下文 → 几万 token。
**方式 B（codemode）**：模型写一段循环脚本 → 沙箱里调 50 次工具 → 只把「结果摘要」带回上下文 → 几百 token。

### 沙箱里有什么

```ts
const sandbox = new CodemodeSandbox({
  timeoutMs: 60_000,
  tools: [{ name: "read", execute: async (args, { signal }) => ... }],
});

const result = await sandbox.execute(`
  const source = await tools.read({ path: "package.json" });
  text("bytes " + source.length);
  return JSON.parse(source).name;
`);
```

脚本能用的全局量（**这就是全部**）：

| 名称 | 作用 |
|---|---|
| `tools.<name>(args)` | 调工具，返回 Promise；名字非法字符变 `_` |
| `ALL_TOOLS` | `{ name, description }[]` |
| `text(value)` | 追加文本输出（非字符串 JSON 化） |
| `image(urlOrItem)` | 追加图片（只接受 base64 data URL） |
| `exit()` | 立刻成功结束 |
| `store(key, value)` / `load(key)` | 同步读写 JSON（跨执行的状态） |
| `console.*` | 等同 `text` |

**没有**：定时器、`fetch`、`process`、`require`、模块、`WebAssembly`。`eval`/`Function` 可以，但只能在同一 VM 里造更多代码。

### 资源限制

- `timeoutMs`：默认有截止时间，`Infinity` 关闭（但 signal 仍可中断）；
- `memoryLimitBytes`：超出 VM 堆 → `InternalError: out of memory`；
- **死锁检测**：如果脚本等一个永远不会有结果的 promise（没有工具在跑、VM 没有定时器/IO），会立刻失败而不是挂住。

最后这条很关键：沙箱里没有 `setTimeout`，所以一个 `await` 永远不 settle 的脚本会永久占用一个 worker。pi 直接把它判为失败。

### `store` / `load` 的持久化模型

沙箱**自己不落盘**：

```ts
const result = await sandbox.execute(code, { store: saved });
if (result.ok) {
  for (const key of result.storeWrites.delete) delete saved[key];
  Object.assign(saved, result.storeWrites.set);
}
```

失败的执行不产生写入——**事务语义**：要么脚本成功并且写入生效，要么什么都不变。

### `// @options:` 行

脚本开头可以指定选项（超时、是否 spread 参数等），让模型对单次执行做细粒度控制。

### 在 coding-agent 里的形态

`src/extensions/codemode/`（6 文件，含 `worker.ts` / `execute.lazy.ts`）：

- 内置一个叫 `codemode` 的工具；
- 脚本里的 `tools.*` 通过 `runToolCall()` 走 agent 的完整管线（第 09 章），所以 `beforeToolCall` / `afterToolCall` **对嵌套调用一样生效**；
- 嵌套调用记录在 `ToolResultMessage.nestedCalls`（有大小上限，超了只留 `argumentsBytes`）。

## 3. 两者怎么选（以及为什么可以共存）

```
任务：查一堆文件的 TODO 注释
├─ 直接工具：grep → 结果（简单，1 次调用）
└─ codemode：写脚本遍历 + 过滤 + 汇总（省上下文，但要写代码）

任务：调用公司内部 API 的 5 个端点
├─ MCP：暴露成 5 个工具（模型直接选）
└─ codemode：暴露成 5 个函数，模型写一段编排（组合逻辑复杂时更省）
```

pi 同时内置两者，并且 `ToolExposure` 让同一个工具既可以被「直接看到」也可以被「代码里调用」：

```ts
export type ToolExposure = "direct" | "model-only" | "codemode" | "deferred" | "hidden";
```

`tool-search` 扩展则解决另一侧问题：工具太多时，让模型先「搜索工具」再加载——`registerTool({ defaultActive: false })` + 激活机制。

**三个机制合起来是「工具经济」的完整方案**：直接调用（贵的但直观）、代码编排（省上下文）、按需发现（省工具列表）。

## 4. 本章要点

- `pi-mcp` 是零依赖 MCP 客户端，核心工程点是 `toLlmContent` 的降级转换与工具名清洗。
- `pi-codemode` 用 QuickJS/WASM 沙箱把「N 次工具调用」压成「1 次脚本」，只有输出进上下文。
- 沙箱能力极窄（无网络/无定时器/无模块），有超时、内存上限、死锁检测、事务式 store。
- 嵌套工具调用走 `runToolCall()`，共享安全钩子，并记录在 `nestedCalls`（有上限）。
- `ToolExposure` + `tool_search` 让「工具很多」不再等于「上下文爆炸」。

下一章：[22 pi-telemetry](22-pi-telemetry.md)
