# 01 ReAct 执行回路：步数限制、自我纠错、工具失败重试

## 一、二开前：今天的完整流程（逐行对应源码）

### 1.1 触发与入口

```
用户输入 → AgentSession.prompt()
        → SessionManager.append(user 消息，先落盘)
        → Agent.prompt()  →  new AbortController() → agentLoop()
```

### 1.2 循环本体（`packages/agent/src/agent-loop.ts` `runLoop`）

```ts
while (true) {                                    // 外层：follow-up 兜底
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

**关键事实：`while (true)` 里没有任何计数器。** 我全仓搜索 `maxSteps|maxTurns|maxIterations|stepLimit|loopGuard` 在 `packages/agent/src` 与 `agent-session.ts` 中**零命中**。

### 1.3 一轮 turn 的判定链

| 顺序 | 动作 | 能否终止循环 |
|---|---|---|
| 1 | `prepareNextTurn` | 否（只能改上下文/模型/追加消息） |
| 2 | `declareToolChanges` | 否 |
| 3 | `prepareRequest` | 否 |
| 4 | `streamAssistantResponse` | **是**：`stopReason` 为 `error`/`aborted` 直接 return |
| 5 | 提取 `toolCall` 并执行 | 否 |
| 6 | `finishTurn` | **是**：返回 `{ action: "end" }` |
| 7 | poll steering，决定 `hasMoreToolCalls` | 是（false 且无 pending 则退出内层） |

**所以今天终止循环只有四条路**：
1. 模型不再发工具调用（`stopReason: "stop"`）；
2. 模型报错/被中断（`error`/`aborted`）；
3. `finishTurn` 判断 `end`；
4. 所有工具结果都 `terminate: true`（`shouldTerminateToolBatch` 全员通过）；
5. 用户 `abort()`。

**第 3 条实际用在哪**：`AgentSession` 的 `finishTurn` 只在「上下文溢出恢复」和「排队消息」场景返回 `continue`，**从不主动 `end`**。也就是说产品层没有基于**轮次数量**的刹车。

### 1.4 工具参数失败时今天怎么处理

`prepareToolCall`（`agent-loop.ts:707`）的 `try/catch`：

```ts
try {
  const preparedToolCall = prepareToolCallArguments(tool, toolCall);   // 可选兼容修复
  const validatedArgs = validateToolArguments(tool, preparedToolCall); // TypeBox 校验
  if (config.beforeToolCall) { ...可 block... }
  return { kind: "prepared", toolCall, tool, args: validatedArgs };
} catch (error) {
  return {                                    // ← 校验失败走这里
    kind: "immediate",
    result: createErrorToolResult(error instanceof Error ? error.message : String(error)),
    isError: true,
  };
}
```

即：**校验失败 → 造一条错误 toolResult → 交给模型自己看着办**。

相关的另一个保护在 `length` 截断：

```ts
const executedToolBatch = message.stopReason === "length"
  ? await failToolCallsFromTruncatedMessage(toolCalls, emit)   // 整批失败
  : await executeToolCalls(...);
```

### 1.5 现有的「重试」到底是什么

`packages/ai/src/utils/retry.ts`：

```ts
export interface RetryPolicy {
  enabled: boolean;
  maxRetries: number;      // 默认 3
  baseDelayMs: number;     // 默认 2000（2s/4s/8s 指数退避）
  maxAgentDelayMs: number; // 默认 60000
}
export function isRetryableAssistantError(message: AssistantMessage): boolean
```

**它重试的是「模型请求失败」**（429、5xx、网络），而**不是「工具调用失败」**。

### 1.6 结论：死循环从哪里来

```
模型发 toolCall(edit, 参数不合法)
  → 校验失败 → 错误 toolResult
  → 模型看到错误，重发同一个调用（参数还是错的）
  → 校验又失败 → ...
  ↑ 没有任何计数器，没有任何相同调用检测，循环可以无限
```

**今天唯一阻止它的是模型自己的判断力和用户的 Esc。** 这正是需求里「死循环」的真实来源——不是网络，是**参数语义错误 + 模型不自省**。

所以第 1 项的三件事其实是同一件事的三个层次：

1. **计步**（兜底，防跑飞）
2. **判重**（识别「同一个错」在重复）
3. **纠错**（把「重复」变成「换个做法」或「停下来问人」）

---

## 二、差距表

| 能力 | 现状 | 缺口 |
|---|---|---|
| 最大轮次 | 无 | 需新增 `maxTurns`，且要可配置、可在恢复时延续 |
| 步数耗尽后的行为 | 无 | 需定义：硬停？转摘要？降级模型？交还用户 |
| 重复调用检测 | 无 | 需 `(toolName, argsHash)` 连续计数 |
| 参数失败重试 | 只有一条错误结果交回模型 | 需**有界**的修复/重试策略 |
| 自我反思 | 无专门机制 | 需在 `finishTurn` 或 `prepareNextTurn` 插入反思步 |
| 反思成本控制 | — | 反思要调模型，本身消耗 token，需计量与上限 |

---

## 三、二开思路

### 3.1 结构选择：不要改 `runLoop` 的骨架

`agent-loop.ts` 是 13 个下游共用的核心，且 `AgentEvent` 序列有严格不变式（每个 `start` 都有 `end`）。**能在 `Agent` 层或 `AgentSession` 层解决的，就不要动内核。**

三种做法对比：

| 方案 | 改哪 | 侵入性 | 推荐 |
|---|---|---|---|
| A. 纯上层 | `AgentSession.finishTurn` 里计数 | 零核心改动 | ✅ **先做这个** |
| B. 内核加计数器 | `agent-loop.ts` 的 `runLoop` | 改核心 | 只在需要「循环内可中断的工具批次」时 |
| C. 新 config 字段 | `AgentLoopConfig` 加 `maxTurns` | 中等 | 若要让所有 agent-core 使用者都受益 |

**建议路线：A 先落地（1 天），验证指标；若证明不够，再升到 C。**

### 3.2 骨架（方案 A）

```ts
// packages/coding-agent/src/core/agent-session.ts 附近

interface LoopGuardState {
  turns: number;
  recentCalls: Array<{ key: string; count: number }>;
  reflectionDone: Set<string>;   // 已对哪些 key 反思过，防止反思本身刷屏
}

const MAX_TURNS = settings.maxTurns ?? 40;              // 可配
const REPEAT_WARN = 3;                                   // 同 key 连续 3 次 → 警告
const REPEAT_HARD = 5;                                   // 连续 5 次 → 硬停

session.agent.finishTurn = async (turn, signal) => {
  guard.turns += 1;

  // ① 硬步数上限
  if (guard.turns >= MAX_TURNS) {
    await session.appendSystemNote(
      `Reached the ${MAX_TURNS}-turn budget. Stop calling tools and summarize what you know.`,
    );
    return { action: "end" };
  }

  // ② 重复调用检测（用 toolResult 的 isError + 参数哈希）
  const repeated = detectRepeatedFailures(turn.toolResults, guard);
  if (repeated.hard) {
    return { action: "end" };      // 或转成"交还用户"
  }

  // ③ 软提示：不是硬停，而是给模型一次自省机会（只做一次）
  if (repeated.soft && !guard.reflectionDone.has(repeated.key)) {
    guard.reflectionDone.add(repeated.key);
    await session.appendSystemNote(reflectionPrompt(repeated));
  }
  return undefined;   // 保持正常调度
};
```

`appendSystemNote` 就是往转录里追加一条 `system` 消息（第 10 章的分节补丁机制），**不需要碰 agent-core**。

### 3.3 反思（Self-Reflection）放在哪：三个候选与取舍

| 位置 | 时机 | 优点 | 缺点 |
|---|---|---|---|
| `finishTurn` 追加 system 消息 | 工具批次结束后 | 简单、不改内核、模型下一轮自然看到 | 反思内容是"提示"而非"结论" |
| `prepareNextTurn` 里调一次模型做反思 | 下轮请求前 | 反思可以引用完整上下文 | **多一次模型调用**，且可能与压缩时序冲突 |
| 失败时立即重试（不改模型） | `prepareToolCall` | 省 token | 只能修语法，修不了语义 |

**推荐组合：`finishTurn` 软提示（免费）+ 参数层修复重试（省一次模型往返）+ 只在硬失败时调模型反思。**

### 3.4 参数失败的重试策略（有界，三层）

```
第 1 层：本地修复（0 次模型调用）
  - 类型宽松转换：字符串数字 → 数字、"true"/"false" → 布尔
  - 去除多余字段、补默认值
  - 工具可声明 `prepareArguments`（已有钩子！）做兼容
  → 若修复后通过校验，直接执行

第 2 层：结构化错误回灌（0 次模型调用）
  - 错误信息里带：哪个字段错、期望什么、收到的什么、正确示例
  - 今天只回灌 error.message（TypeBox 的原始报错，模型常看不懂）
  → 这是最高性价比的改动：把"报错"改成"可执行的纠错说明"

第 3 层：有界重试（1 次模型调用）
  - 同 key 第 2 次失败：追加一条带 schema 摘要的 system 消息
  - 同 key 第 3 次失败：强制换成"让模型用自然语言描述想做什么"
  - 第 4 次：硬停 + 交还用户
```

**注意已有钩子**：`AgentTool.prepareArguments?: (args: unknown) => Static<TParameters>` 就是为「参数兼容修复」预留的，且在校验**之前**调用（`agent-loop.ts:693`）。第 1 层可以**完全不动核心**，只在各工具的 `prepareArguments` 里实现。

---

## 四、二开后的完整流程

```
prompt()
  → 落盘 user 消息
  → agentLoop()

  turn N
   ├─ prepareNextTurn
   │    └─【新】检查 guard.turns，接近上限时提前在 system 消息里降级提示
   ├─ prepareRequest
   ├─ streamAssistantResponse
   └─ 工具批次
        ├─ prepareToolCall
        │    ├─【新】第 1 层：prepareArguments 本地修复
        │    ├─【新】第 2 层：结构化错误（字段级、带示例）
        │    └─ 失败 → error toolResult
        ├─ execute
        └─ finishTurn
             ├─【新】turns += 1
             ├─【新】turns ≥ maxTurns → 追加收尾指令 + {action:"end"}
             ├─【新】重复失败检测（key 连续计数）
             │     ├─ soft（3 次）→ 追加反思提示（每 key 仅一次）
             │     └─ hard（5 次）→ end，并在会话里标注原因
             └─ 正常 → undefined（调度不变）

  → agent_end
  → 落盘 + 统计（新增：turns、重复次数、反思次数，写进 usage/自定义 entry）
```

**可观测性**：新增三个指标落进会话文件的自定义 entry，`scripts/session-transcripts.ts` 可离线统计，直接回答「死循环率降了多少」。

---

## 五、工作量、风险、验收

| 项 | 人日 |
|---|---|
| 步数上限 + 配置 + 落盘指标 | 1.5 |
| 重复检测（哈希、连续计数、阈值） | 2 |
| 结构化错误（改 `createErrorToolResult` 调用点，需要看 TypeBox 报错格式） | 2 |
| `prepareArguments` 在各内置工具上的实现 | 2 |
| 反思提示词设计与调参 | 2 |
| 测试（新增 suite + faux provider） | 1.5 |
| **合计** | **约 8–11** |

### 风险

| 风险 | 说明 | 缓解 |
|---|---|---|
| **反思自身变成循环** | 每次失败都反思 → 每轮多一条 system 消息 → 上下文膨胀 → 触发压缩 → 更多轮次 | 每个 key 只反思一次；反思内容短（<200 token）；计入 maxTurns |
| 阈值误伤 | 合法场景也会重复调用（轮询等待构建完成） | 阈值可配；`read`/`ls` 这类只读工具豁免；只看「连续失败」不看「连续调用」 |
| 硬停体验差 | 复杂任务被 40 轮截断 | 硬停前先让模型出摘要，会话仍可 continue |
| 改 `finishTurn` 与既有溢出恢复冲突 | `AgentSession.finishTurn` 已有逻辑 | 用 wrapper 链式包裹，不要替换（参考 `agent-session.ts:1721` 对 `transformContext` 的链式做法） |

### 验收

1. **死循环率**：用 `scripts/session-transcripts.ts` 导出改造前后各 ≥50 个会话，按「同一 `(tool,argsHash)` 连续 ≥5 次」统计，对比下降幅度。
2. **参数失败自愈率**：埋点统计「第 1 层修复后通过校验」的比例。这是纯赚的指标。
3. **不回归**：`coding-agent/test/suite/` 用 faux provider 跑一遍完整场景（含工具调用），确认事件序列不变。
