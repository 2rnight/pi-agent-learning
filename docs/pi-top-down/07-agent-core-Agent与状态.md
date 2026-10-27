# 07 Agent 与状态（pi-agent-core）

`pi-agent-core` 只有 2513 行，却是整个仓库杠杆率最高的地方：

```
src/agent.ts        613 行   Agent 类：状态 + 队列 + 事件分发
src/agent-loop.ts   940 行   循环本身（纯函数 + 事件 sink）
src/types.ts        529 行   所有类型契约
src/proxy.ts        406 行   浏览器代理流
src/stream-fn.ts     20 行   默认 streamFn 注入点
```

## 1. 为什么既有 `Agent` 又有 `agentLoop`

这是 pi 一个刻意的分层：

| | `agentLoop()` | `Agent` |
|---|---|---|
| 形态 | async generator，纯函数 | 有状态类 |
| 输入 | `(messages, context, config, signal, streamFn)` | 构造函数 + 方法 |
| 输出 | `AsyncIterable<AgentEvent>` | 事件订阅 + Promise |
| 是否等待消费者 | **否**（observational） | **是**（barrier） |
| 适用 | 无 UI 的批处理、测试、代理转发 | 真实产品（pi CLI 就用它） |

`packages/agent/README.md` 明确写了这个差别：

> These low-level streams are observational. They preserve event order, but they do not wait for your async event handling to settle before later producer phases continue. If you need message processing to act as a barrier before tool preflight, use the `Agent` class.

**为什么 barrier 很重要**：工具执行前，agent 需要「assistant 消息已经进入状态」。如果 UI 还在异步处理 `message_end`（比如落盘、渲染），而工具已经开始跑，就会出现「工具结果先于请求它的消息落盘」的乱序。`Agent` 通过 `await` 每个监听者来保证顺序。

## 2. `AgentState`：哪些是可变，哪些是只读

```ts
export interface AgentState {
  readonly systemPrompt: string;   // 只读：从转录重放
  model: Model<any>;
  thinkingLevel: ThinkingLevel;
  set tools(tools: AgentTool<any>[]);   // 赋值时复制顶层数组
  get tools(): AgentTool<any>[];
  set messages(messages: AgentMessage[]);
  get messages(): AgentMessage[];
  readonly isStreaming: boolean;
  readonly streamingMessage?: AgentMessage;
  readonly pendingToolCalls: ReadonlySet<string>;
  readonly errorMessage?: string;
}
```

三个值得注意的点：

1. **`systemPrompt` 是只读的**。想改提示词？往 `messages` 里追加一条 `system` 消息（带 `content` 或 `sections`）。这就是第 04 章说的「转录即真相」。

2. **`tools` / `messages` 的 setter 会复制顶层数组**：

   ```ts
   agent.state.tools = [...]      // 外部数组被复制，之后外部改动不影响 agent
   agent.state.messages.push(m)   // 但 getter 返回的是内部数组，push 会生效
   ```

   注释写得很明确：*Assigning ... copies the top-level array before storing it.* 这是「防止外部持有引用后乱改」的折中——浅拷贝既避免别名，又保留原地 append 的性能。

3. **`isStreaming` 的语义比名字长**：它一直为 `true`，直到**所有 `agent_end` 监听者都结算完毕**。所以 `await agent.waitForIdle()` 才真正代表「可以安全读盘」的状态。

## 3. 事件：9 种，分三层

```ts
export type AgentEvent =
  // agent 生命周期
  | { type: "agent_start" }
  | { type: "agent_end"; messages: AgentMessage[] }
  // turn 生命周期（一轮 = 一次模型响应 + 它的工具调用/结果）
  | { type: "turn_start" }
  | { type: "turn_end"; message: AgentMessage; toolResults: ToolResultMessage[] }
  // 消息生命周期
  | { type: "message_start"; message: AgentMessage }
  | { type: "message_update"; message: AgentMessage; assistantMessageEvent: AssistantMessageEvent }  // 仅 assistant
  | { type: "message_end"; message: AgentMessage }
  // 工具生命周期
  | { type: "tool_execution_start"; toolCallId; toolName; args }
  | { type: "tool_execution_update"; toolCallId; toolName; args; partialResult }
  | { type: "tool_execution_end"; toolCallId; toolName; result; isError };
```

注意 `message_update` **只对 assistant 消息发**——user/toolResult 是原子产生的，没有流式过程。

这套事件的消费方式在上层分叉（第 19 章）：
- `InteractiveMode` → 更新 TUI 组件；
- `print-mode` → 只关心 `message_end` 的文本；
- RPC → 序列化成 JSONL；
- `AgentSession` → 落盘 + 统计 + 通知扩展。

## 4. 两个队列：steering 与 follow-up

这是 pi 交互体验的核心机制，值得单独理解。

```ts
agent.steer(msg)     // 塞进 steeringQueue：当前 assistant turn 的工具跑完就注入
agent.followUp(msg)  // 塞进 followUpQueue：agent 本来要停了才注入
```

区别在于**注入时机**：

```
turn N: assistant 说要调 3 个工具
   ├─ 工具 1/2/3 执行
   ├─ finishTurn
   ├─ turn_start (N+1)
   ├─ ← steering 在这里注入（"停下，改做这个"）
   └─ 下一次请求包含了 steering

... 没有工具调用、没有 steering 了
   ├─ 检查 follow-up 队列
   └─ ← follow-up 在这里注入（"顺便再总结一下"）
```

`QueueMode` 控制每次注入几条：`"one-at-a-time"`（默认）或 `"all"`。

**为什么默认 one-at-a-time**：用户可能连打三句话，但模型应该一条一条消化，而不是被三句话混合成一个含糊的请求。`peekQueuedMessages()` 可以让 UI 提前显示「下一条会是什么」而不消费它。

## 5. 可替换字段一览（`Agent` 的「旋钮」）

`Agent` 暴露的全部钩子（从 `agent.ts` 的字段声明直接读出来）：

| 字段 | 时机 | 用途 |
|---|---|---|
| `streamFunction` | 每次请求 | 换模型传输方式（真实 / 代理 / 假） |
| `convertToLlm` | 每次请求 | `AgentMessage[] → Message[]` |
| `transformContext` | 每次请求，`convertToLlm` 之前 | 裁剪、注入 |
| `getApiKey` | 每次请求 | 动态 token |
| `onPayload` / `onResponse` | 请求前后 | 审计、改写请求体 |
| `onProviderStreamEvent` | 流式过程中 | 观测原始 provider 事件 |
| `beforeToolCall` | 每个工具执行前 | 阻止危险操作 |
| `afterToolCall` | 每个工具执行后 | 改写结果 |
| `finishTurn` | 每轮结束 | 决定继续/结束 |
| `prepareRequest` | 每次请求前 | 装入权威上下文 |
| `prepareNextTurn` / `prepareNextTurnWithContext` | 轮次之间 | 压缩、切模型、注入消息 |
| `toolExecution` | 运行时 | 并行/串行 |
| `sessionId` | 构造后 | 供应商侧缓存亲和性 |
| `thinkingBudgets` | 运行时 | 按档位配 token 预算 |
| `transport` | 运行时 | `sse` / `websocket` / `auto` |

**这张表就是 pi 的可扩展性清单**。你想改行为，先看表里有没有现成插入点，再考虑改内核。

## 6. 状态复制与重置

```ts
agent.reset();              // 清空上下文（保留配置）
agent.abort();              // 中断当前 run
await agent.waitForIdle();  // 等 run + agent_end 监听者结算
```

`abort()` 与「队列」的交互值得注意（README 有说明）：**中断后队列里的消息会回到编辑器**，不会丢。这是「用户按 Esc 不该丢输入」的体验设计。

## 7. 一个最小可用的 agent

README 的 Quick Start 浓缩起来就是这样：

```ts
const agent = new Agent({
  initialState: { systemPrompt: "You are a helpful assistant.", model },
  streamFn: models.streamSimple.bind(models),
});

agent.subscribe((event) => {
  if (event.type === "message_update" && event.assistantMessageEvent.type === "text_delta") {
    process.stdout.write(event.assistantMessageEvent.delta);
  }
});

await agent.prompt("Hello!");
```

注意 `streamFn: models.streamSimple.bind(models)`——**agent-core 不知道 pi-ai 的注册表存在**，它只要求一个符合 `StreamFn` 形状的函数。这是依赖倒置的具体体现。

## 8. 本章要点

- `agentLoop` 是纯生成器（不等待消费者）；`Agent` 是有状态门面（等待监听者，提供 barrier）。
- `systemPrompt` 只读，真相在转录里；`tools`/`messages` 赋值会复制顶层数组。
- 两种队列对应两种时机：steering 在 turn 之间，follow-up 在「本来要停」时。
- 15 个可替换字段就是 pi 的扩展点清单，改行为优先找插入点。

下一章：[08 agent-loop 循环](08-agent-core-agent-loop.md)
