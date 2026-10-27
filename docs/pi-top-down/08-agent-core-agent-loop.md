# 08 agent-loop 循环

`packages/agent/src/agent-loop.ts`（940 行）是整个 pi 的「心跳」。这一章把它的结构彻底拆开。

## 1. 对外只有四个函数

```ts
export function agentLoop(messages, context, config, signal?, streamFn?): AgentEventStream
export function agentLoopContinue(context, config, signal?, streamFn?): AgentEventStream
export async function runAgentLoop(...): Promise<void>       // 内部：带 emit 回调
export async function runAgentLoopContinue(...): Promise<void>
```

前两个返回生成器（给低层用户），后两个是内部实现（给 `Agent` 类传 `emit` sink 用）。

真正的核心是 `runLoop()`。

## 2. 双层循环：内层处理工具/steering，外层处理 follow-up

```ts
while (true) {                                  // 外层：准备「本来要停了」
  let hasMoreToolCalls = true;

  while (hasMoreToolCalls || pendingMessages.length > 0) {   // 内层：正常推进
    ...
  }

  const followUpMessages = await config.getFollowUpMessages?.() ?? [];
  if (followUpMessages.length > 0) { pendingMessages = followUpMessages; continue; }

  if (explicitContinuation) { explicitContinuation = false; continue; }

  break;
}
```

**为什么用两层**：内层是「有事情要做就继续做」（工具结果、steering），外层是「没事做了，但用户又排了新活」。两层分工让 `finishTurn` 的 `continue` 语义很干净——它只需要标记「还要再来一次」，而不必关心是谁促成的。

## 3. 一轮 turn 的完整顺序（逐行对应源码）

下面是 `runLoop` 内层循环体的真实顺序，编号即执行顺序：

| # | 动作 | 源码位置 | 说明 |
|---|---|---|---|
| 1 | `prepareNextTurn(lastCompletedTurn)` | 循环开头 | 只有「已有完成轮次」时才调；可改 context/model/thinkingLevel，可返回要追加的消息 |
| 2 | 再次 poll steering | 紧接着 | 若步骤 1 耗时长（压缩可能要几十秒），期间用户可能又插话 |
| 3 | `emit turn_start` | | 注意：第一次 turn 不发（`lastCompletedTurn` 为空） |
| 4 | `declareToolChanges(context, prepared + pending)` | | 把「可执行工具集」的变化写成 system 消息的 `toolsAdded/toolsRemoved` |
| 5 | 逐条 `message_start` + `message_end` + push | | pending 消息在这里正式进入转录 |
| 6 | `prepareRequest({context, model, thinkingLevel})` | | 最后一次换上下文的机会 |
| 7 | `streamAssistantResponse(...)` | | 调模型、发 `message_*` 事件 |
| 8 | **`error`/`aborted` → 收尾并 return** | | 硬退出：先 `finishTurn`，再 `turn_end`，再 `agent_end` |
| 9 | 提取 `toolCalls` | | `content.filter(c => c.type === "toolCall")` |
| 10 | `stopReason === "length"` → `failToolCallsFromTruncatedMessage` | | 否则 `executeToolCalls` |
| 11 | 工具结果 push 进 context 与 newMessages | | **注意是循环外统一 push** |
| 12 | `finishTurn(lastCompletedTurn)` | | 返回 `{action:"continue"} \| {action:"end"} \| undefined` |
| 13 | `emit turn_end` | | 在 finishTurn 之后 |
| 14 | `action === "end"` → `agent_end` + return | | 不 poll 队列 |
| 15 | poll steering，决定是否继续 | | 有工具结果或有 steering 就继续 |

有两点容易读错：

- **`turn_start` 不在第一轮发**。第一轮直接进入消息处理（因为 `agent_start` 已经标志开始了）。
- **`finishTurn` 在 `turn_end` 之前跑**，所以它可以阻止后续排队，但阻止不了 `turn_end`（那一轮确实结束了）。

## 4. `declareToolChanges`：工具集变化怎么告诉模型

这是 pi 里比较独特的设计。模型看到的工具集是**系统消息里声明的**，而运行时能执行的工具是 `context.tools`。两者可能不一致（扩展动态加工具、工具搜索按需启用、loadout 切换）。

```ts
function declareToolChanges(context: AgentContext, pendingMessages: AgentMessage[]): AgentMessage[] {
  // 找到 pending 里最后一条 system 消息
  // 若存在：它的 tools 字段视为「意图」，用「已提交转录 vs 当前可执行集」的差量替换
  // 若不存在：在第一条非 system 的 pending 消息之前插入一条新的 system 消息
}
```

**为什么这么绕**：为了让「重放转录」永远得到精确的 `context.tools`。如果简单地在每次请求前都 insert 一条新 system 消息，会话文件会爆；如果直接改已有消息，历史就不可变了。

pi 的解法：**把差量合并进已有的 pending system 消息**，保证「按序重放 = 当前状态」，同时不污染已提交的历史。

## 5. `streamAssistantResponse`：60 行干三件事

前面章节已经展示过代码，这里总结它的三个职责：

1. **两次上下文转换**：`transformContext` → `convertToLlm` → `normalizeContext`；
2. **每轮解析 key**（`getApiKey`），支持 token 过期；
3. **把事件流翻译成 agent 事件**，并且保证 `context.messages` 最后一项始终是最新的部分消息。

一个隐藏细节：

```ts
const result = async () => Object.assign(await response.result(), { thinkingLevel: config.reasoning ?? "off" });
```

`Object.assign` 给最终消息**盖上一个 `thinkingLevel` 标记**——因为不同 stream function 可能不填这个字段，但事后归因（「这次回答用的是 high 还是 medium？」）必须有它。

## 6. 退出路径只有四条

在一个容易出 bug 的循环里，pi 把退出条件收敛得很干净：

```
① streamAssistantResponse 返回 stopReason = error/aborted   → turn_end + agent_end
② finishTurn 返回 { action: "end" }                        → turn_end + agent_end
③ 没有工具调用 + 没有 steering + 没有 follow-up + 无 continue → break → agent_end
④ 调用方 abort（signal）                                    → 工具循环里 break，流里报 aborted
```

**`agent_end` 是唯一出口**：所有路径在 return 前都 `await emit({type:"agent_end", messages:newMessages})`。这保证了「subscriber 收到 `agent_end` 就一定不会再有本 run 的事件」。

## 7. `newMessages` 的语义

`agent_end.messages` 里放的是**本次 run 新产生的消息**，不是整个转录。

```ts
export interface AgentTurnContext {
  message: AssistantMessage;
  toolResults: ToolResultMessage[];
  context: AgentContext;
  /** 这次 loop 调用若在此退出会返回的消息。prompt run 含初始 prompt 消息；continuation run 不含既有上下文。 */
  newMessages: AgentMessage[];
}
```

这个区分对 `AgentSession` 很重要：它只需要把「新消息」写盘，而不是把整个转录重写一遍。

## 8. 一个完整的 trace（带工具调用）

```
agent_start
turn_start                                  ← 注意：实际是第二轮才有
message_start/end  (user: "读 config.json")
message_start      (assistant, streaming...)
message_update × N (text_delta)
message_end        (assistant, stopReason: "toolUse", 1 个 toolCall)
tool_execution_start  (id=call_1, name=read, args={path:"config.json"})
tool_execution_update (可选，工具自己流式上报)
tool_execution_end    (result, isError=false)
message_start/end  (toolResult)
turn_end           (message, toolResults:[...])
turn_start
message_start ... message_end  (assistant, stopReason: "stop")
turn_end
agent_end          (messages: [user, assistant, toolResult, assistant])
```

对照 README 的图，你会发现 README 把第一个 `turn_start` 画在了 `agent_start` 之后——那是 `Agent` 类在 `prompt()` 时补的语义，和 `runLoop` 内部条件发送有细微差别。**以源码为准**。

## 9. 本章要点

- **双层循环**：内层推进（工具/steering），外层兜底（follow-up / explicit continue）。
- 一轮内的顺序是：`prepareNextTurn → 注入消息 → 声明工具变化 → prepareRequest → 请求 → 工具 → finishTurn → turn_end`。
- `declareToolChanges` 保证「重放转录 = 精确工具集」，是 pi 状态一致性的关键。
- 只有四条退出路径，且只有 `agent_end` 一个出口。
- `agent_end.messages` 只含本次 run 的新消息。

下一章：[09 工具执行管线](09-agent-core-工具执行管线.md)
