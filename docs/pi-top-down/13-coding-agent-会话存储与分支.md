# 13 会话存储与分支树

pi 的会话不是「一个消息数组」，而是**一棵树**。这一章讲为什么、怎么存、怎么把树投影成「模型该看的历史」。

## 1. 文件格式：JSONL + 树

位置（`docs/session-format.md`）：

```
~/.pi/agent/sessions/--<path>--/<timestamp>_<session-id>.jsonl
```

第一行是 header，之后每一行是一个 entry：

```jsonc
{"type":"session","version":3,"id":"uuid","timestamp":"...","cwd":"/path/to/project"}
{"type":"message","id":"a1b2c3d4","parentId":null,"timestamp":"...","message":{...}}
{"type":"message","id":"e5f6a7b8","parentId":"a1b2c3d4","timestamp":"...","message":{...}}
```

`SessionEntryBase`：

```ts
interface SessionEntryBase {
  type: string;
  id: string;              // 通常是 8 位 hex，必要时退回完整 UUID
  parentId: string | null; // null = 根
  timestamp: string;       // ISO 8601（注意：消息体内的 timestamp 是 Unix ms）
}
```

**追加写 + parentId 指针**带来三个直接能力：

1. **分支无需新文件**：从任意 entry 继续，新 entry 的 `parentId` 指向它，树就分叉了。
2. **崩溃安全**：追加写不破坏已有内容（最坏情况是半行，加载时可跳过）。
3. **可读性**：`tail -f` 就能看会话在长。

## 2. Entry 类型：不止是消息

`session-manager.ts` 定义的完整集合（这就是「会话记录了什么」）：

| 类型 | 接口 | 作用 |
|---|---|---|
| `message` | `SessionMessageEntry` | 一条 `AgentMessage` |
| `model_change` | `ModelChangeEntry` | 何时换了模型 |
| `thinking_level_change` | `ThinkingLevelChangeEntry` | 何时换了思考档位 |
| `usage` | `UsageEntry` | token/成本累计 |
| `compaction` | `CompactionEntry` | 压缩摘要 + `firstKeptEntryId` |
| `branch_summary` | `BranchSummaryEntry` | 切分支时对旧分支的摘要 |
| `custom` | `CustomEntry` | 扩展自定义数据 |
| `custom_message` | `CustomMessageEntry` | 扩展注入的消息 |
| `label` | `LabelEntry` | 用户给某个节点起的名字 |
| `session_info` | `SessionInfoEntry` | 会话元信息（名字等） |
| `context_edit` | `ContextEditEntry` | 手动编辑上下文（删/改某条） |

**关键观察**：模型切换、压缩、扩展状态、用户标签**都在同一棵树里**。所以「恢复到某个时间点」= 把 active leaf 移到那个 entry，一切状态自动还原（包括当时用的模型）。

## 3. 三个版本与迁移

```
v1: 线性 entry 序列（legacy）
v2: 树结构（id/parentId）
v3: hookMessage 改名为 custom（扩展统一）
```

`CURRENT_SESSION_VERSION = 3`。加载时 `migrateSessionEntries()` 自动升级，`parseSessionEntries()` 容错解析。**不做向后兼容的分支判断，而是一次性迁移到最新版**——这符合 `AGENTS.md` 的原则「Do not preserve backward compatibility unless the user asks」。

## 4. 从树到上下文：三层投影

这是最容易混淆的部分。源码里有名字相近的四个函数，用途完全不同：

```
buildSessionProjection(entries)      树 → 投影（带 active branch、压缩替换后的条目序列）
       │
       ├─ buildContextEntries(projection)         投影 → 该进上下文的条目
       │
       └─ buildSessionContext(...)                条目 → SessionContext（含 model/thinking/usage）
       
sessionEntryToContextMessages(entry)   单个 entry → AgentMessage[]
getLatestCompactionEntry(entries)      找最近一次压缩（作为重放起点）
```

### 压缩如何「替换」历史

`CompactionEntry` 有 `firstKeptEntryId`。投影规则：

```
if 存在 compaction entry:
    上下文 = [摘要消息] + （firstKeptEntryId 及其之后的所有条目）
else:
    上下文 = 从根到 active leaf 的全部条目
```

**原始条目一个都没删**——只是不再参与投影。所以「压缩后还能回看完整历史」是天然成立的，这也让「压缩后悔」成为可能。

## 5. 分支操作：四种，语义不同

| 操作 | 行为 | 文件 |
|---|---|---|
| **继续**（回复历史节点） | 同一文件内新增分支，active leaf 移到新 entry | 同一个 `.jsonl` |
| **fork** | 复制选中历史到**新文件**，`parentSession` 字段指回原文件 | 新 `.jsonl` |
| **clone** | 类似 fork，但通常指「整段复制」而非「在某点分叉」 | 新 `.jsonl` |
| **branch summary** | 切分支时，把离开的那段历史压成摘要带过去 | 写入 `branch_summary` entry |

header 里的 `parentSession`：

```json
{"type":"session","version":3,"id":"...","cwd":"...","parentSession":"/path/to/original/session.jsonl"}
```

这让「会话家族」可追溯（`/fork`、`/clone`、`newSession({ parentSession })` 都会写这个字段）。

## 6. `SessionManager`：树的所有者

`session-manager.ts`（2013 行）对外是一个类，核心职责：

- 加载/解析/迁移；
- 维护 `activeLeafId`；
- `append(entry)`（带自动 id/parentId）；
- 提供投影（给 `AgentSession.prepareRequest` 用）；
- 列表/搜索/最近会话（`findMostRecentSession`、`SessionListProgress`）；
- 清理与删除（互动删除时优先用 `trash` CLI 而不是永久删除）。

SDK 文档有一句很重要的话：

> `SessionManager` is authoritative for finalized model context. Restore external history by constructing the session with a manager containing those entries. Assigning `session.agent.state.messages` does not replace persisted context.

翻译：**你改 agent 内存没用，真相在 manager 里**。这解释了为什么需要 `prepareRequest`（第 12 章）。

## 7. 为什么用树而不是线性 + 分支文件

对比一下常见做法：

| 方案 | 优点 | 缺点 |
|---|---|---|
| 分支 = 新文件 | 简单、隔离 | 文件爆炸、跨分支 diff 难 |
| 线性 + 「回到某点重写」 | 简单 | 丢历史、不能并排比较 |
| **单文件树（pi 的选择）** | 一次写入、历史完整、切换 O(1) | 文件会变大、需要投影逻辑 |

pi 的投影层（`buildSessionProjection`）就是把这种存储优势兑换成「上下文正确」的成本。**收益**：`/tree` 可以可视化整棵树、任意跳转、任意分叉，而不用管文件。

## 8. 会话文件的其他消费者

- `session-export.ts` / `export-html/`：把会话导出成 HTML（带工具渲染）；
- `session-cwd.ts`：处理「会话记录的 cwd 已经不存在」的情况（`MissingSessionCwdError`），提示用户选择新路径；
- `usage-totals.ts`：从 `usage` entry 汇总成本；
- `cache-stats.ts`：命中率统计；
- `evals/scripts/session-transcripts.ts`：把会话转成评测语料。

**这棵树是整个产品的数据底座**，几乎所有「历史相关」功能都建立在它之上。

## 9. 本章要点

- 会话 = 单文件 JSONL + `id/parentId` 树；追加写、崩溃安全、可 `tail`。
- entry 类型覆盖消息、模型切换、压缩、分支摘要、扩展数据、标签、上下文编辑。
- 上下文由**投影**产生：压缩用 `firstKeptEntryId` 换掉旧段，但原始 entry 永不删除。
- `SessionManager` 是模型上下文的唯一权威；改内存不生效。
- 树存储 + 投影层 = 任意分支/回看能力的来源。

下一章：[14 内置工具集](14-coding-agent-内置工具集.md)
