# 10 上下文装配

模型每次请求只能看到「一个消息数组」。pi 的挑战是：它在内存/磁盘里维护的是**更丰富的 `AgentMessage[]`**（含 UI 通知、bash 执行记录、扩展自定义角色），必须转换、裁剪、补丁成模型能懂的东西。

这一章讲清 `transformContext` → `convertToLlm` → `SystemMessage` 补丁 这条装配线。

## 1. 为什么需要两段转换

```
AgentMessage[]                        （应用态：可以有任意角色）
      │  transformContext()           （可选，纯 AgentMessage 级操作）
      ▼
AgentMessage[]                        （裁剪后、注入后，仍是应用态）
      │  convertToLlm()               （必须，转成模型态）
      ▼
Message[]                             （只有 system/user/assistant/toolResult）
      │  normalizeContext()
      ▼
TranscriptContext → streamFn
```

两个函数的**契约**在 `types.ts` 里写得很死：

```ts
/** Contract: must not throw or reject. Return the original messages or another safe fallback value instead. */
transformContext?: (messages: AgentMessage[], signal?: AbortSignal) => Promise<AgentMessage[]>;

/** Contract: must not throw or reject. Return a safe fallback value instead. */
convertToLlm: (messages: AgentMessage[]) => Message[] | Promise<Message[]>;
```

**为什么禁止抛错**：这两个函数在循环的咽喉位置。一旦抛错，事件序列就会断裂（有 `message_start` 没有 `message_end`），UI 卡死。返回安全兜底值（哪怕是原始数组）至少能继续跑。

这条「不抛错契约」在 pi 里反复出现：`StreamFn`、`transformContext`、`convertToLlm`、`getSteeringMessages`、`getFollowUpMessages` 都有。**这是一个统一的设计原则：中间层回调失败时必须退化，不能中断协议。**

## 2. `transformContext`：在 AgentMessage 层动手

适合做的事（README 明确列出）：

- **上下文窗口管理**（剪掉旧消息）；
- **注入外部上下文**（比如把某个文件内容临时加进去）。

不适合做的事：把消息转成另一种角色——那是 `convertToLlm` 的活。

典型实现（README 例子）：

```ts
transformContext: async (messages, signal) => {
  if (estimateTokens(messages) > MAX_TOKENS) return pruneOldMessages(messages);
  return messages;
}
```

注意：pi 真实产品里，**主动裁剪主要交给 compaction（第 16 章）**，`transformContext` 更多用于扩展注入。

## 3. `convertToLlm`：过滤 + 降级

默认实现（`agent.ts` 的 `defaultConvertToLlm`）基本就是「只保留已知的四种角色」。真实产品的 `convertToLlm` 在 `packages/coding-agent/src/core/messages.ts`，要处理这些自定义角色：

| 自定义角色 | 转换方式 |
|---|---|
| `bashExecution` | 变成一段文本（user 或 toolResult 形态） |
| `custom`（扩展消息） | 按扩展声明处理，或过滤 |
| `notification` 等 UI-only | **直接丢弃** |
| `compaction` 摘要 | 变成一条 system 或 user 消息 |

README 的映射例子：

```ts
convertToLlm: (messages) => messages.flatMap((m) => {
  if (m.role === "custom") return [{ role: "user", content: m.content, timestamp: m.timestamp }];
  if (m.role === "notification") return [];       // UI-only，丢掉
  return [m];
});
```

**这里有个容易踩的坑**：丢弃消息会改变「模型看到的历史」，但**不会改变会话文件**。所以同一段会话在不同版本 pi 下重放，模型看到的内容可能不同——这是设计选择（会话记录是「发生了什么」，上下文是「这次给模型看什么」）。

## 4. 系统提示词：pi 最不寻常的状态设计

传统 agent 框架：`state.systemPrompt = "..."`，改了就改全局。

pi：**提示词是转录的一部分，通过 system 消息重放得到**。

```ts
export interface SystemMessage {
  role: "system";
  content: string | TextContent[];
  sections?: Record<string, string | null>;  // 具名分节补丁
  toolsAdded?: Tool[];
  toolsRemoved?: ToolReference[];
  timestamp: number;
}
```

### 三种修改方式

| 想做的事 | 做法 |
|---|---|
| 追加一段新指令 | `append system message with content:"..."` |
| 替换某个具名分节（如 `<skills>`） | `append system message with sections:{ skills: "<skills>...</skills>" }` |
| 删除某个分节 | `sections: { skills: null }` |

README 的例子：

```ts
await agent.prompt([
  { role: "system", content: "", sections: { skills: "<skills>...</skills>" }, timestamp: Date.now() },
  { role: "user", content: "Continue", timestamp: Date.now() },
]);
```

### 为什么值得这么做

1. **分支天然正确**：会话树里每个节点都有自己的 system 消息序列，分叉不用「重算提示词」。
2. **提示词变化可审计**：什么时候加了什么技能，都在转录里。
3. **压缩/回放安全**：投影到某个历史点，提示词状态自动正确。
4. **扩展友好**：扩展在 `before_agent_start` 里追加分节即可，不用改全局状态。

代价是：**读「当前提示词」需要一次重放**。pi-ai 提供了 `getCurrentSystemMessage(messages)`，`agent.state.systemPrompt` 就是它的结果（只读）。

## 5. 工具集声明：`toolsAdded` / `toolsRemoved`

这是第 8 章提到的 `declareToolChanges` 的配套设计。

```
运行时真相：context.tools（能执行什么）
模型认知：转录里 system 消息累积声明的工具集（能调什么）
```

二者必须一致，否则会出现「模型调用了一个它以为不存在的工具」或者「工具存在但模型不知道」。

pi 的做法：

- **首次请求**：第一条 system 消息里带完整 `tools`；
- **之后变化**：追加 system 消息，只带 `toolsAdded` / `toolsRemoved` 的差量；
- **pending 消息里已有 system 消息**：把差量合并进去（视为「意图」），而不是插新消息。

注释里点明了目标：

> Replaying them in order yields the current prompt and tools; there is no separate prompt state entry.

这个「纯重放」特性对 `coding-agent` 极其重要——第 13 章的会话文件就是这么存的。

## 6. `normalizeContext`

最后一步（`agent-loop.ts`）：

```ts
const llmContext = normalizeContext({ messages: llmMessages });
```

`normalizeContext` 来自 `pi-ai`，负责把消息数组整理成供应商无关的规范形态。它存在的意义是：**provider 实现可以假设输入已经规整**，不必每个 api 文件都写一遍「补 timestamp / 合并不合法的相邻消息 / 清理空 content」。

实现很短，但有两个设计点（`packages/ai/src/utils/transcript.ts`）：

```ts
const transcriptContextBrand: unique symbol;

export type TranscriptContext = {
  messages: Message[];
  readonly [transcriptContextBrand]: true;   // 只有 normalizeContext() 能造出来
};

export function normalizeContext(context: Context): TranscriptContext {
  const initialMessage = createInitialSystemMessage(context.systemPrompt, context.tools);
  const messages = initialMessage ? [initialMessage, ...context.messages] : context.messages;
  return { messages } as TranscriptContext;
}
```

1. **品牌类型（branded type）**：`TranscriptContext` 带一个只读 symbol 标记，注释写着 "Only `normalizeContext()` produces this type, so a raw `Context` cannot reach provider code by accident"。**用类型系统防止「忘了规范化就传给 provider」**——这是一个很值得学的技巧。
2. **兼容旧形态**：`normalizeContext` 会把传入的 `Context.systemPrompt` / `Context.tools` **折叠成一条开头的 system 消息**。也就是说，即使调用方还在用「context 上挂 systemPrompt」的旧写法，进入 provider 前也会被统一成「转录里带 system 消息」的新形态。

配套的还有 `api/transform-messages.ts`：把统一消息转换成特定方言（比如 Anthropic 要求 tool_result 在 user 消息里、OpenAI 要求 tool 消息紧跟 assistant）。

## 7. `TranscriptContext` 与「系统提示词不在 context 上」

一个容易困惑的点：`StreamFn` 的签名是

```ts
type StreamFn = (model, context: TranscriptContext, options?) => ...;
```

而文档反复强调：

> The loop passes a normalized transcript: the system prompt and tool declarations are carried by the transcript's system messages, never by `context.systemPrompt` or `context.tools`.

也就是说：**`TranscriptContext` 只有 `messages`**，没有独立的 `systemPrompt` 字段。这是「转录即真相」在类型层面的强制。

## 8. 装配顺序总结（一图）

```
agent.state.messages (AgentMessage[])
   │
   ├─ ① transformContext      （裁剪 / 注入外部上下文）
   ├─ ② convertToLlm          （过滤 UI-only，降级自定义角色）
   ├─ ③ normalizeContext      （规整成供应商无关 Message[]）
   ├─ ④ api/transform-messages（转成特定方言的请求体）
   ▼
HTTP/WS 请求
```

四个阶段分别属于：**应用 / 应用 / 库 / 协议**。分清这个归属，就能知道一个改动的正确落点在哪。

## 9. 本章要点

- 两段转换 + 两次规整：`transformContext` → `convertToLlm` → `normalizeContext` → 方言转换。
- 所有中间回调**禁止抛错**，必须退化——保证事件序列完整。
- 系统提示词不是独立状态，而是 system 消息的**顺序重放**；改提示词 = 追加重放补丁。
- 工具集声明也是 system 消息（`toolsAdded/toolsRemoved`），与转录重放绑定。
- 「模型看到什么」和「磁盘记录了什么」是两个问题，`convertToLlm` 是它们的分界线。

下一章：[11 coding-agent 分层架构](11-coding-agent-分层架构.md)
