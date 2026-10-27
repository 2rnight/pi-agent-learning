# 12 AgentSession：把 agent-core 变成产品

`core/agent-session.ts`（4349 行）是 `coding-agent` 的心脏。它的文件头注释列出了全部职责：

```ts
/**
 * AgentSession - Core abstraction for agent lifecycle and session management.
 *
 * This class is shared between all run modes (interactive, print, rpc).
 * It encapsulates:
 * - Agent state access
 * - Event subscription with automatic session persistence
 * - Model and thinking level management
 * - Compaction (manual and auto)
 * - Bash execution
 * - Session switching and branching
 *
 * Modes use this class and add their own I/O layer on top.
 */
```

一句话：**`Agent` 是「会跑」的，`AgentSession` 是「会被记住、被压缩、被切换、被扩展影响」的。**

## 1. 它解决了 agent-core 不管的四件事

| 问题 | agent-core 的答案 | AgentSession 的答案 |
|---|---|---|
| 消息存哪？ | 不管（内存数组） | 每条消息落 JSONL 树 |
| 上下文太长？ | 提供 `transformContext` 钩子 | 实现压缩（自动 + `/compact`） |
| 换模型/换会话？ | 不管 | model registry + session 切换/分叉 |
| 扩展怎么介入？ | 不给概念 | ExtensionRunner 广播 30+ 种事件 |

## 2. 事件订阅 + 自动持久化

最关键的一段逻辑：`AgentSession` 订阅底层 `Agent` 的事件，**在事件处理里写盘**。

```
Agent 事件                        AgentSession 的处理
────────────────────────────────────────────────────────
message_end (user)      →  append SessionMessageEntry 到会话文件
message_end (assistant) →  append + 累计 usage
message_end (toolResult)→  append
model 变化              →  append ModelChangeEntry
thinkingLevel 变化      →  append ThinkingLevelChangeEntry
压缩完成                →  append CompactionEntry
工具/会话事件           →  转发给 ExtensionRunner
```

**为什么放在事件里而不是 `prompt()` 的返回值里**：因为一次 run 会产出任意多消息（多轮 + 多工具），只有事件流知道全部。而且 UI 崩溃、用户中断时，已经落盘的部分仍然可恢复。

这也解释了第 07 章为什么强调 `Agent` 会 `await` 监听者：**落盘是监听者做的，必须保证顺序**。

## 3. `prepareRequest`：权威上下文注入点

这是 `AgentSession` 用到的第一个高级钩子。它把「会话文件的投影」塞回 agent：

```ts
// 概念示意
agent.prepareRequest = async ({ context }) => ({
  context: { ...context, messages: await loadCanonicalMessages() },
});
```

**为什么需要**：agent 内存里的 `messages` 可能包含

- UI-only 的自定义消息（不进模型上下文）；
- 已经被压缩摘要替代的旧消息（会话文件里有，但不该再发）；
- 扩展临时追加的内容。

而「应该发给模型的精确历史」由 `SessionManager` 的投影（projection）计算。`prepareRequest` 是两者对齐的最后时机。

## 4. `finishTurn` / `prepareNextTurn`：产品级调度

| 钩子 | AgentSession 用它做什么 |
|---|---|
| `prepareNextTurn` | **自动压缩检查**（工具结果落盘后、下一次请求前）、切换模型、注入分支摘要 |
| `finishTurn` | 判断是否要因「上下文溢出」或「输出被截断」做恢复重试，返回 `{action:"continue"}` |
| `beforeToolCall` | 触发扩展的 tool-call 事件（可被拦截）、工具准入 |
| `afterToolCall` | 触发扩展结果事件、规范化图片、截断超长输出 |

`docs/compaction.md` 对压缩时机的描述正好对应这里的实现：

> During a multi-turn agent run, Pi checks the canonical projected context after tools finish and their results are appended, before starting the next assistant response. If the threshold is crossed, Pi compacts during `prepareNextTurn`...

**这就是「压缩不是定时器，而是挂在轮次边界上的检查」**——只有在「工具结果都进来了、还没发下一次请求」这个时刻，上下文长度才是明确且稳定的。

## 5. 一个会话对象内部的组成

```
AgentSession
├── agent: Agent                    (agent-core)
├── sessionManager: SessionManager  (持久化 + 投影)
├── settingsManager                 (global/project 设置)
├── extensionRunner                 (扩展事件总线)
├── modelRuntime / modelRegistry    (模型解析与凭证)
├── resourceLoader                  (skills / prompts / extensions 发现)
├── 工具集合                         (createCodingTools() + 扩展注册)
├── steering / followUp 队列         (经 agent)
├── compaction 状态                 (auto compaction abort controller 等)
└── 统计                            (usage totals、cache stats、timings)
```

从源码里可以看到很具体的东西：`_compactionAbortController`、`_autoCompactionAbortController`、`_branchSummaryAbortController`——**三种摘要各自一个可取消的控制器**，因为用户可能在压缩中途按 Esc。

## 6. 对外 API 的形状

SDK 文档给出的读法：

```ts
const { session } = await createAgentSession();
await session.prompt("...");
session.messages;          // 当前上下文
session.model;             // 当前模型
session.thinkingLevel;
session.systemPrompt;      // 只读，有效提示词
session.getActiveToolNames();
session.dispose();
```

写操作的核心方法（从源码签名归纳）：

| 方法 | 作用 |
|---|---|
| `prompt(text, options?)` | 发一条用户输入 |
| `abort()` | 中断当前 run（同时中断压缩） |
| `compact(customInstructions?)` | 手动压缩（`/compact` 走这里） |
| `reload()` | 重载 settings + 资源（`/reload` 走这里，扩展热替换） |
| `newSession()` / `fork()` / `switchSession()` | 会话生命周期 |
| `revert`/`tree` 相关 | 会话树导航（第 13 章） |

`reload()` 的实现很直接：

```ts
await this.settingsManager.reload();
await this._resourceLoader.reload();
// …重建扩展运行时
```

但文档里反复警告一个坑：

> Reload replaces the extension runtime, so code after `await ctx.reload()` must not reuse state from the old runtime.

源码里还有一条更狠的错误信息（`agent-session.ts:1375`）：

> This extension ctx is stale after session replacement or reload. Do not use a captured `pi` or command ctx after `ctx.newSession()`, `ctx.fork()`, `ctx.switchSession()`, or `ctx.reload()`.

**这是在用运行时检查弥补 TypeScript 类型表达不了的「对象生命周期」**。扩展作者必须用 `withSession` 传入的新 ctx。

## 7. 为什么这个类会膨胀到 4349 行

因为它承担了「产品粘合层」的全部负担，而粘合层天然横跨所有维度：

```
维度：模式（4种）× 会话（新建/恢复/分叉/克隆）× 模型（切换/重试）×
      上下文（压缩/裁剪/摘要）× 扩展（30+ 事件）× 工具（8个）×
      凭证（登录/刷新/过期）× 错误（重试/溢出/中断）
```

这些不是「功能」，而是**每两个维度的交叉都要有正确的行为**。例如：在压缩期间来了 steering 消息怎么办？在 rpc 模式下扩展想弹对话框怎么办？工具执行中被 abort 后 toolResult 该写什么？

`AgentSession` 就是这些交叉的答案集合。这也是为什么 `modes/` 可以很薄——复杂性被上收到了这里（以及 `interactive-mode.ts`）。

## 8. 与 SDK 的关系

`core/sdk.ts` 的 `createAgentSession()`（175 行起）是唯一推荐入口：

```ts
export interface CreateAgentSessionOptions { /* 40+ 字段 */ }
export interface CreateAgentSessionResult { session; /* … */ }
```

它做的事：合并默认值 → 建立 auth/settings/model runtime → 建 resource loader → 建 tools → 建 session manager → new AgentSession → 应用扩展。

**读 `sdk.ts` 是理解「一个 pi 会话由多少个部件拼成」最快的方式**（比读 `agent-session.ts` 快得多）。

## 9. 本章要点

- `AgentSession` = agent-core + 持久化 + 压缩 + 扩展 + 模型管理，四个运行模式共用。
- 它通过**订阅 Agent 事件**来落盘，所以 `Agent` 必须 await 监听者。
- `prepareRequest` 注入权威上下文，`prepareNextTurn` 做自动压缩，`finishTurn` 做恢复决策。
- 4349 行的根源是「维度交叉」而非「功能堆积」，所以它难以拆分。
- `sdk.ts` 是理解装配的最佳入口；`reload`/`withSession` 的对象生命周期是扩展的最大陷阱。

下一章：[13 会话存储与分支树](13-coding-agent-会话存储与分支.md)
