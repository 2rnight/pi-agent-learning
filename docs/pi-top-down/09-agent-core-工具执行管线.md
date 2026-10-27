# 09 工具执行管线

「模型说要用工具」到「结果回到转录」之间有一整条流水线。pi 把它拆成 **准备 → 执行 → 终结** 三个阶段，并允许在阶段之间插入钩子。

## 1. 管线全景

```
assistant message.content 里的 toolCall 块
        │
        ▼
┌───────────────────────┐
│ prepareToolCall       │  ① 找工具（找不到→immediate error）
│  - prepareArguments   │  ② 兼容性参数修复（可选）
│  - 参数 schema 校验    │  ③ 校验失败→immediate error
│  - beforeToolCall     │  ④ 可 block（→immediate error，可带 terminate）
└───────────┬───────────┘
            │ kind: "prepared" | "immediate"
            ▼
┌───────────────────────┐
│ executePreparedToolCall│  execute(id, params, signal, onUpdate)
│  - 捕获 throw          │  抛错 → error 结果（message 进 content）
└───────────┬───────────┘
            ▼
┌───────────────────────┐
│ finalizeExecutedToolCall│  afterToolCall 钩子，可逐字段覆盖
│  - content/details/    │  无深合并：给了就整体替换
│    isError/usage/      │
│    terminate           │
└───────────┬───────────┘
            ▼
     emitToolExecutionEnd  →  createToolResultMessage  →  emitToolResultMessage
```

## 2. `prepareToolCall` 的四个出口

```ts
type PreparedToolCall = { kind: "prepared"; toolCall; tool; args };
type ImmediateToolCallOutcome = { kind: "immediate"; result; isError };
```

「immediate」意味着**不执行工具，直接产出一个结果**。四种情况会走这条路：

| 情况 | 结果 |
|---|---|
| 工具名不存在 | error tool result，提示模型工具不可用 |
| `prepareArguments` 抛错 | error |
| schema 校验失败 | error |
| `beforeToolCall` 返回 `{ block: true }` | error（`reason` 作为文本，可带 `terminate`） |

**为什么要区分 immediate**：被拒绝的调用仍然要产生一条 `toolResultMessage`。模型需要看到「你调用了但没成功」，否则它会以为工具调用丢了，可能无限重试或干脆放弃。

## 3. 参数校验

工具参数用 **TypeBox**（`typebox` 依赖，`Type.Object({...})`）定义 JSON Schema，校验发生在执行前：

```ts
export interface AgentTool<TParameters extends TSchema, TDetails> extends Tool<TParameters> {
  label: string;
  prepareArguments?: (args: unknown) => Static<TParameters>;  // 兼容层
  outputSchema?: TSchema;
  execute: (toolCallId, params, signal, onUpdate) => Promise<AgentToolResult<TDetails>>;
  replay?: "never" | "safe";
  executionMode?: ToolExecutionMode;
}
```

- **`prepareArguments`**：模型/供应商有时给的参数结构略有偏差（比如数字给成字符串）。这个钩子在**校验之前**做兼容修复。它必须返回符合 schema 的对象。
- **`outputSchema`**：描述 `structuredContent` 的形状，用于程序化调用者（不发给模型）。
- **`replay`**：持久化重放策略——`"never"` 表示有副作用不能自动重放，`"safe"` 表示幂等。这是给 durable 那套「崩溃恢复」用的（第 25 章）。
- **`executionMode`**：单个工具可以强制整批串行。

## 4. 并行 vs 串行：真实实现

```ts
const hasSequentialToolCall = toolCalls.some(
  (tc) => currentContext.tools?.find((t) => t.name === tc.name)?.executionMode === "sequential",
);
if (config.toolExecution === "sequential" || hasSequentialToolCall) {
  return executeToolCallsSequential(...);
}
return executeToolCallsParallel(...);
```

**注意这个保守规则**：只要批里**任何一个**工具声明了 `executionMode: "sequential"`，整批都串行。不是「那个工具单独串行」，而是降级整批。

为什么这么设计？因为并行执行时你无法保证顺序——如果 `edit` 是串行的、`read` 是并行的，同时执行它们会在读到一个正在被写的文件。**整批降级是可预测性的代价最小解。**

### 并行实现的一个巧妙之处

`executeToolCallsParallel` 并不是「先全部 prepare，再 `Promise.all` 执行」那么简单：

```ts
finalizedCalls.push(async () => {       // 把「执行」包成闭包
  if (signal?.aborted) return abortedResult;   // 执行前检查 abort
  const executed = await executePreparedToolCall(...);
  const finalized = await finalizeExecutedToolCall(...);
  await emitToolExecutionEnd(finalized, emit);   // 完成就发，不等别人
  return finalized;
});
...
const orderedFinalizedCalls = await Promise.all(finalizedCalls.map(e => typeof e === "function" ? e() : ...));
```

- **prepare 是串行的**（每个 toolCall 依次签名/校验/hook），这样 `beforeToolCall` 看到的 agent 状态是一致的；
- **execute 是并发的**；
- **`tool_execution_end` 按完成顺序发**（谁先完成谁先发事件，UI 可以立刻显示）；
- **但 toolResult 消息按 assistant 源顺序发**（`orderedFinalizedCalls` 保序），保证转录稳定。

这个「事件顺序 ≠ 消息顺序」的二分，是 pi 并行工具在 UI 上不混乱、在存储上不抖动的原因。

## 5. `terminate`：提前结束的集体决策

```ts
function shouldTerminateToolBatch(finalizedCalls) {
  return finalizedCalls.length > 0 && finalizedCalls.every((f) => f.result.terminate === true);
}
```

**全员否决制**：只有批里**每一个**工具结果都说 `terminate: true`，才跳过「再问一次模型」的自动调用。

三个来源可以设 terminate：`execute()` 的返回值、被 block 的 `beforeToolCall`、`afterToolCall`。

为什么是「全体」而不是「任一」？因为一个批里通常有「真正干活」和「顺便查一下」两类工具；前者想收工，后者没必要拦着整个 run。混合批继续跑是更安全的默认。

代价：`terminate` **不写进转录**（`toolResultMessage` 是标准 LLM 格式），只是运行时提示。所以重放不会得到同样的终止行为——这是刻意的。

## 6. 工具自己也能流式

`execute()` 的第 4 个参数：

```ts
export type AgentToolUpdateCallback<T> = (partialResult: AgentToolResult<T>) => void;
```

工具可以在执行中多次 `onUpdate(partial)`，pi 会转成 `tool_execution_update` 事件。用途：`bash` 命令的实时输出、长搜索的进度、`codemode` 脚本里的嵌套调用进度。

一个契约细节：**回调是「绑定到本次 execute 调用」的**，工具 promise 结束后再调 `onUpdate` 会被忽略（避免泄漏到下一次执行）。

## 7. 错误处理：抛错 vs `isError`

README 给了明确规则：

```ts
// 推荐：抛错
execute: async (id, params) => {
  if (!fs.existsSync(params.path)) throw new Error(`File not found: ${params.path}`);
  return { content: [{ type: "text", text: "..." }] };
}
```

- **抛错** → pi 捕获，生成 `isError: true` 的结果，`content` 是错误信息；
- **返回 `isError: true`** → 保留 `details` 和 `structuredContent`（用于 UI 展示与程序断言），同时让模型看到 `content` 里的错误文本。

两者对模型是一样的（都是错误结果），区别在于**给程序/UI 的信息量**。所以：纯失败用 throw，需要携带结构化失败信息用 `isError`。

## 8. `runToolCall`：脱离循环单独调用工具

```ts
export async function runToolCall(toolCall, options: RunToolCallOptions): Promise<AgentToolCallOutcome>
```

这个导出很重要：`codemode` 沙箱里模型写的 JS 调用工具时，走的就是 `runToolCall()`。**好处是 `beforeToolCall` / `afterToolCall` 一样生效**——嵌套调用不会绕过安全钩子。

`packages/agent/examples/mcp-codemode/` 就是这两种集成的示范：MCP 工具一个一个包成 `AgentTool`；`codemode` 工具则通过 `runToolCall` 拿到 agent 的完整工具集。

## 9. 截断保护：`length` 时整批失败

```ts
const executedToolBatch = message.stopReason === "length"
  ? await failToolCallsFromTruncatedMessage(toolCalls, emit)
  : await executeToolCalls(...);
```

注释解释了原因：流式工具调用的参数是用「尽力而为的 JSON 抢救解析器」拼出来的，被截断的消息**可能产生能通过校验但语义残缺的参数**。执行这种调用的风险远大于重试成本。

这是全文最值得记住的「生产级细节」之一。

## 10. 本章要点

- 管线三段：prepare（校验+hook，串行）→ execute（并发）→ finalize（afterToolCall 覆盖）。
- 被拒绝/校验失败的调用依然产生 toolResult，模型才能自我纠正。
- 整批串行的触发条件是「任一工具声明串行」，为了可预测性。
- 事件按完成顺序、消息按源顺序——两个顺序是分开设计的。
- `terminate` 全员通过才生效，且不持久化。
- `runToolCall()` 让嵌套（codemode）调用共享同一套钩子。

下一章：[10 上下文装配](10-agent-core-上下文装配.md)
