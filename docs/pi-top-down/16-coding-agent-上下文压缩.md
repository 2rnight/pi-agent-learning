# 16 上下文压缩（Compaction）

长会话必然撞上下文窗口。pi 有**两种摘要机制**，都在 `core/compaction/`：

| 机制 | 触发 | 目的 |
|---|---|---|
| **Compaction** | 上下文超阈值，或 `/compact` | 把旧消息换成摘要，腾出空间 |
| **Branch summarization** | `/tree` 切换分支 | 换分支时保留旧分支的要点 |

## 1. 触发条件：一个不等式

```ts
export interface CompactionSettings {
  enabled: boolean;
  reserveTokens: number;
  keepRecentTokens: number;
}

export const DEFAULT_COMPACTION_SETTINGS: CompactionSettings = {
  enabled: true,
  reserveTokens: 16384,
  keepRecentTokens: 20000,
};

export function shouldCompact(contextTokens, contextWindow, settings): boolean {
  if (!settings.enabled) return false;
  return contextTokens > contextWindow - settings.reserveTokens;
}
```

即：**留 16k 给「模型的回复 + 下一次请求的余量」，剩下的都可以用**。

## 2. 触发时机：挂在轮次边界上

`docs/compaction.md` 描述得很精确：

> During a multi-turn agent run, Pi checks the canonical projected context **after tools finish and their results are appended, before starting the next assistant response**. If the threshold is crossed, Pi compacts during `prepareNextTurn`, then performs the existing catch-up steering poll before `turn_start`.

为什么是这一刻？因为只有此时：
- 上下文是稳定的（没有半截流式消息）；
- 工具结果都在（不会压缩完又冒出个大文件）；
- 还没发下次请求（来得及改）。

其他触发点：

- **新用户 prompt 之前**；
- **provider 报上下文溢出错误后**：一次「压缩并重试」的恢复（`isContextOverflow`）；
- **`stopReason: "length"`（提前截断）**：也可能触发一次恢复尝试；
- **手动 `/compact [instructions]`**。

`agent-session.ts` 里有三个独立的 abort controller（`_compactionAbortController`、`_autoCompactionAbortController`、`_branchSummaryAbortController`）——用户可以在压缩中途按 Esc。

## 3. Token 估算：先粗后精

pi 用两条路算 token：

1. **精确**：最后一条 assistant 消息的 `usage` 里有真实 token 数（`calculateContextTokens`）。但之后新增的消息没有 usage。
2. **估算**：chars/4 启发式（`estimateTokens`），注释说 "conservative (overestimates tokens)"。

```ts
export function estimateTokens(message: AgentMessage): number {
  // system: content + 所有 sections + toolsAdded 的 JSON 长度
  // user:   文本长度 + 图片固定 4800 字符
  // assistant: text + thinking + (toolCall 名字 + 参数 JSON)
  // 全部 ÷ 4 向上取整
}
```

`compaction.ts` 里有一个聪明的优化（`calculateContextTokens` 附近）：如果最新的 usage 条目比最近一次「context_edit / compaction」还新，就直接返回 usage 估算，不重算整段历史。

**图片按 4800 字符估**：因为图片的 token 成本与分辨率相关，pi 用一个保守常数避免低估。

## 4. 五步流程

```
① 找切点（findCutPoint）
   从最新的 entry 往前累积 token，累积到 keepRecentTokens(20k) 为止
   → 得到一个「保留边界」

② 取消息
   从上一次保留边界（或会话开始）到切点之间的消息 = 待摘要

③ 生成摘要（generateSummary）
   调 LLM，用 SUMMARIZATION_SYSTEM_PROMPT

④ 追加 CompactionEntry
   { summary, firstKeptEntryId, ... }

⑤ 重建上下文
   下一次投影 = [摘要] + firstKeptEntryId 之后的条目
```

示意（来自文档）：

```
压缩前：
 entry: 0    1    2    3     4    5    6     7     8    9
       hdr  usr  ass  tool  usr  ass  tool  tool  ass  tool
             └──── 待摘要 ────┘ └────── 保留 ──────┘
                            ↑
                       切点/首次保留
压缩后：
       hdr … [compaction entry] … 从 firstKeptEntryId 继续
```

**原始 entry 不删除**，只是在投影时被摘要替代。所以 `/tree` 还能看到全部历史。

## 5. 摘要提示词：严格的结构化输出

```ts
export const SUMMARIZATION_SYSTEM_PROMPT = `You are a context summarization assistant.
Your task is to read a conversation between a user and an AI assistant, then produce a structured summary following the exact format specified.

Do NOT continue the conversation. Do NOT respond to any questions in the conversation.
ONLY output the structured summary.`;
```

两个要点：

- **结构化格式**（不是自由文本），便于迭代压缩时「把上次摘要作为上下文再压缩」；
- **明确禁止继续对话**——不加这句，模型很容易「接着聊」，摘要就变成了新的助手回复。

## 6. 文件操作追踪：压缩不能丢失「改了哪些文件」

压缩的经典事故：摘要说「我们修了 bug」，但忘了具体改了哪些文件，导致后续工作推倒重来。

pi 的解法是**累积追踪文件操作**：

```ts
computeFileLists(...)                 // 从消息里提取 read/write/edit 涉及的文件
createFileOps(...) / extractFileOpsFromMessage(...)
formatFileOperations(...)             // 渲染成摘要的一部分
```

也就是说：摘要是「叙事 + 文件清单」两部分，文件清单跨多次压缩累积（每次带上前一次的清单）。这是很实用的工程细节。

## 7. 提示词缓存：一次性请求要关掉缓存写

`docs/compaction.md`：

> Summarization requests disable prompt-cache writes because these one-off prompts are unlikely to be reused.

压缩请求是一次性的（长上下文 + 固定提示词），写缓存既浪费钱又污染缓存命中率。对应 `StreamOptions` / `SimpleStreamOptions` 里的缓存相关选项（`promptCache`、`CacheRetention`）。

## 8. 分支摘要：另一种摘要

`branch-summarization.ts`。场景：用户在 `/tree` 里跳到一条旧分支继续，但「离开的那条分支」上的工作不应该完全丢失。

```ts
const contextWindow = model.contextWindow || 128000;
const tokenBudget = contextWindow - reserveTokens;
```

它生成 `BranchSummaryEntry` 写在新分支上。**和 compaction 的区别**：

| | Compaction | Branch summarization |
|---|---|---|
| 目的 | 腾空间 | 保留另一条分支的要点 |
| 结果 | 替换旧历史 | 追加一条摘要 |
| 触发 | 阈值/命令 | 切分支 |

两者共用 `compaction/utils.ts` 的序列化与文件追踪代码。

## 9. 压缩失败怎么办

`SessionCompactFailedEvent` 存在说明压缩是会失败的（模型报错、被中断、请求超限）。pi 的处理：

- 失败**不破坏会话**（只是没有新 entry）；
- 发出 `session_compact_failed` 事件（扩展/UI 可提示）；
- 如果是「溢出恢复」路径失败，run 以 error 结束，但会话仍然可恢复。

## 10. 本章要点

- 触发条件是一个不等式：`contextTokens > contextWindow - reserveTokens`，默认留 16k、保留最近 20k。
- 时机挂在「工具结果落盘后、下次请求前」（`prepareNextTurn`），因为此时上下文稳定。
- 流程：找切点 → 取消息 → LLM 摘要 → 写 `CompactionEntry(firstKeptEntryId)` → 投影时替换。
- 摘要严格结构化 + 累积文件清单 + 关闭缓存写。
- 原始条目永不删除，压缩是**投影层**的行为。

下一章：[17 扩展机制](17-coding-agent-扩展机制.md)
