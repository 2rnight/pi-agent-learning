# 04 消息与流式事件模型

pi 的类型系统里，**消息**和**事件**是两套东西：

- **消息（Message）** 是「对话历史」，会被持久化、被重放、被发给模型；
- **事件（AssistantMessageEvent）** 是「一次回复的流式过程」，只在内存里存在一瞬间。

理解这个区分，就理解了为什么 pi 能在流式输出的同时保持可恢复的历史。

## 1. 只有四种消息

`packages/ai/src/types.ts`：

```ts
export type Message = SystemMessage | UserMessage | AssistantMessage | ToolResultMessage;
```

| 角色 | 关键字段 | 说明 |
|---|---|---|
| `system` | `content`, `sections?`, `toolsAdded?`, `toolsRemoved?` | **不只是提示词**，还是「提示词补丁」和「工具集声明」 |
| `user` | `content: string \| (TextContent\|ImageContent)[]` | 支持图片 |
| `assistant` | `content`, `api`, `provider`, `model`, `usage`, `stopReason`, `thinkingLevel?` | 记录是谁、用什么档位回答的 |
| `toolResult` | `toolCallId`, `toolName`, `content`, `details?`, `isError`, `nestedCalls?` | 结果回灌给模型 |

### 内容块（content blocks）

```ts
TextContent      { type: "text"; text }
ImageContent     { type: "image"; data /* base64 */; mimeType }
ThinkingContent  { type: "thinking"; thinking; thinkingSignature?; redacted? }
ToolCall         { type: "toolCall"; id; name; arguments /* JSON */ }
```

两个不起眼但重要的字段：

- **`textSignature` / `thinkingSignature`**：供应商特有的「重放数据」（比如 Anthropic 要求把 thinking 块原样带回去）。pi 把它们当作**不透明字符串**保存和回传——这是「无损支持推理模型」的关键。
- **`redacted: true`** 的 thinking 块可能没有可读文本，但保留加密 payload，仍然能重放。

### `SystemMessage` 是 pi 最有特色的设计

```ts
export interface SystemMessage {
  role: "system";
  content: string | TextContent[];
  sections?: Record<string, string | null>;   // 具名分节，后写覆盖前写，null 表示删除
  toolsAdded?: Tool[];                        // 从此刻起可用的工具完整定义
  toolsRemoved?: ToolReference[];             // 从此刻起下线的工具
  timestamp: number;
}
```

注释写得很清楚：**「按顺序重放所有 system message 就得到当前的提示词和工具集」**。也就是说：

- 没有「当前系统提示词」这个独立状态；
- 改变提示词 = 追加一条 system message；
- 改变工具集 = 追加一条带 `toolsAdded/toolsRemoved` 的 system message。

这样带来的能力：会话可以回放到任意历史点、分支天然正确、`fork` 不需要「重算提示词」。

> 为什么 `sections` 的名字不能是数字？注释里写了：JSON 对象会重排整数键，导致顺序不稳定。

## 2. `AssistantMessage` 记录了「怎么答的」

```ts
export interface AssistantMessage {
  role: "assistant";
  content: (TextContent | ThinkingContent | ToolCall)[];
  api: Api; provider: ProviderId; model: string;
  responseModel?: string;      // 供应商实际用的模型（可能和请求不同）
  providerThinkingLevel?: string;  // 供应商原始档位
  thinkingLevel?: ModelThinkingLevel; // pi 请求的档位
  usage: Usage;
  stopReason: StopReason;
  diagnostics?: AssistantMessageDiagnostic[];
  errorMessage?: string;
  rawStopReason?: string;
  endTurn?: boolean;
  timestamp: number;
}
```

`stopReason` 是整个控制流的开关：

```ts
type StopReason = "pending" | "stop" | "length" | "toolUse" | "error" | "aborted" | "deferred";
```

- `toolUse` → 循环继续（要执行工具并再问一次）；
- `length` → 输出被截断（工具参数可能残缺 → 整批失败）；
- `error` / `aborted` → **循环硬退出**（`agent-loop.ts` 里直接 return）；
- `deferred` → 延迟响应（某些供应商的异步任务），配合 `DeferredHandle`。

**注意**：`usage` 在每一次 assistant 消息上都记录，压缩、统计、成本计算全都依赖它。

## 3. 流式事件：12 种事件，三段结构

```ts
export type AssistantMessageEvent =
  | { type: "start"; partial: AssistantMessage }
  | { type: "text_start" | "text_delta" | "text_end"; contentIndex: number; ... ; partial }
  | { type: "thinking_start" | "thinking_delta" | "thinking_end"; contentIndex: number; ...; partial }
  | { type: "toolcall_start" | "toolcall_delta"; contentIndex: number; ...; partial }
  | { type: "toolcall_end"; contentIndex: number; toolCall: ToolCall; partial }
  | { type: "done"; reason: "stop"|"length"|"toolUse"|"deferred"; message }
  | { type: "error"; reason: "aborted"|"error"; error: AssistantMessage };
```

设计要点：

1. **每个事件都带 `partial`**：一个完整的「目前长什么样」的对象。这样 UI 不需要自己累积状态，也不会因为漏掉某个 delta 而错乱。
   类型注释提醒了一个重要区别：`partial` 是**共享的活动对象**（"shared live response-so-far helper, not an event-time snapshot"），会随流继续被更新。所以消费方要么立刻读完，要么自己拷贝 —— `agent-loop.ts` 在发 `message_update` 时做的正是浅拷贝 `{ ...partialMessage }`。
2. **`contentIndex`**：一段回复可能同时有思考、文本、工具调用，index 告诉你在第几个内容块里。
3. **`*_start / *_delta / *_end` 三段式**：`start` 用于开一个新块（UI 可以先把空块放出来），`end` 给出完整内容（用于校验/替换）。
4. **终态只有两个**：`done` 或 `error`。流一定以其中一个结束。

## 4. 事件怎么变成消息

`agent-loop.ts` 的 `streamAssistantResponse()` 做了一件事：**让 `context.messages` 的最后一格始终是「最新的部分消息」**。

```ts
case "start":
  partialMessage = event.partial;
  context.messages.push(partialMessage);   // 占位
  await emit({ type: "message_start", message: { ...partialMessage } });
  break;

case "text_delta": // 以及其它 delta/start/end
  partialMessage = event.partial;
  context.messages[context.messages.length - 1] = partialMessage;  // 原地替换
  await emit({ type: "message_update", assistantMessageEvent: event, message: { ...partialMessage } });
  break;

case "done":
case "error": {
  const finalMessage = await result();     // 从流里取最终消息，并盖上 thinkingLevel
  context.messages[last] = finalMessage;
  await emit({ type: "message_end", message: finalMessage });
  return finalMessage;
}
```

三个细节值得记：

- **`result()` 会补上 `thinkingLevel`**：不管哪个 stream function 回答的，都记录「请求的档位」，方便事后归因。
- **浅拷贝（`{ ...partialMessage }`）**：避免监听者在渲染时读到被下一帧改写的对象。
- **事件 sink 是 `await` 的**：`emit` 返回 Promise，UI 的更新在下一步之前完成（这也是「`Agent` 类被推荐用来做 barrier」的原因，见第 07 章）。

## 5. 工具结果消息的特殊之处

```ts
export type ToolResultMessage<TDetails> = {
  role: "toolResult";
  toolCallId: string;
  toolName: string;
  content: (TextContent | ImageContent)[];   // 模型看到的
  details?: JsonRepresentation<TDetails>;    // 不发给模型，给 UI / 程序调用者
  usage?: Usage;                             // 工具自己的用量
  nestedCalls?: NestedToolCalls;             // 工具内部又调了别的工具（codemode）
  isError: boolean;
};
```

- `content` 是**唯一发送给模型的部分**；
- `details` 是给 UI 渲染彩色 diff、给程序做断言的；
- `nestedCalls` 有大小上限（超出时 `arguments` 省略、只留 `argumentsBytes`），防止嵌套调用记录撑爆会话文件。

## 6. 类型层面的两个硬约束

### 约束一：`Message` 必须可序列化成 JSON

`packages/ai/src/types.ts` 里对 `ToolResultMessage<TDetails>` 用了条件类型：

```ts
export type ToolResultMessage<TDetails = JsonValue> =
  IsJsonCompatible<TDetails> extends true ? { ... } : never;
```

意思是：**`details` 不是 JSON 兼容类型时，这个类型直接变成 `never`**，编译期就报错。这保证了会话文件永远可写、可传、可回放。

### 约束二：`AgentMessage` 可扩展，但 `Message` 不可

`packages/agent/src/types.ts`：

```ts
export interface CustomAgentMessages {
  // 默认空，应用通过 declaration merging 扩展
}
export type AgentMessage = Message | CustomAgentMessages[keyof CustomAgentMessages];
```

- `AgentMessage` 可以有 `notification`、`bashExecution`、`custom` 等角色；
- 但发给模型前必须经过 `convertToLlm` 过滤成 4 种；
- 这就是第 10 章「上下文装配」要处理的核心问题。

## 7. 本章要点

- **消息 = 历史（持久化），事件 = 过程（一次性）**；两者都有严格的不变式。
- `SystemMessage` 同时承担提示词、提示词补丁、工具集声明三种角色，靠「按序重放」得到当前状态。
- 每个流式事件都带完整 `partial` 快照，UI 不需要自己拼装。
- `stopReason` 决定循环是否继续；`error`/`aborted` 是硬退出。
- `ToolResultMessage` 严格区分「给模型看的 content」和「给程序看的 details」。

下一章：[05 Provider 与模型目录](05-pi-ai-Provider与模型目录.md)
