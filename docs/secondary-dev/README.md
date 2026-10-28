# pi 二次开发可行性评估（四项需求）

> 目标仓库：`2rnight/pi-agent-learning`（commit `5b6c792` + 本次文档提交）
> 评估对象：① ReAct 循环与自我纠错 ② 分层记忆（Redis + Faiss） ③ 工具沙箱与语义路由 ④ 全双工流式与状态同步
> 评估方法：先读源码确认「二开前」的真实流程与可插入点，再判断「二开后」要改哪里、代价多大。

---

## 0. 先说结论（三个可能影响排期的判断）

### 判断一：四项里有三项「仓库已有一半」，直接开工会重复造轮子

| 需求 | 仓库里已有的东西 | 位置 |
|---|---|---|
| 动态步数限制 | 无。`agent-loop.ts` 是 `while (true)`，**零计数器、零死循环检测** | `packages/agent/src/agent-loop.ts:179` |
| 工具异常重试 | 有重试，但**只针对模型供应商报错**（`maxRetries: 3`，指数退避） | `packages/ai/src/utils/retry.ts`、`settings-manager.ts:46` |
| 参数解析失败处理 | 有「截断时整批失败」保护，但**校验失败只产出一条错误结果，不修复不重试** | `agent-loop.ts:707` `prepareToolCall` |
| 记忆 | 会话树 + 投影 + 压缩 + 分支摘要，本身就是分层记忆 | `session-manager.ts`、`core/compaction/` |
| 语义路由 | **`ToolRanker` 接口已定义，注释直接写「BM25 today; a hybrid ranker with embeddings can replace it」** | `extensions/tool-search/tool.ts` |
| 工具沙箱 | QuickJS/WASM + `worker_threads`，每次执行新开 worker | `packages/codemode/src/runtime/host.ts` |
| 全双工流式 | COT（`thinking_delta`）与工具状态**本来就是流式透传**；打断（`abort`）与队列回填已有 | `agent-loop.ts:381`、`agent.ts` |
| 双向协议 | CBOR 信封 + 任意有序字节传输 + 多客户端附着 + 带外状态 | `protocol` / `client` / `server` |

**这意味着：四项需求的正确起手式不是写代码，而是「先盘点已有什么，再定义缺口」。** 本目录每篇文档的第一节都是「二开前全流程」，就是为了钉死这一点。

### 判断二：三个指标需要先校准口径，否则做完了也验收不了

| 指标 | 问题 | 建议口径 |
|---|---|---|
| 死循环率降低 90% | 目前**没有「死循环」的度量**，也就没有基线 | 用会话 JSONL 离线算：同一 `(toolName, argsHash)` 连续出现 ≥N 次，或连续 M 轮无状态变更（无文件写、无新 tool 名）。`scripts/session-transcripts.ts` 可直接批量导出语料 |
| 候选工具检索耗时降低 60% | **当前 BM25 是进程内内存计算，单次微秒级**。再降 60% 意义接近于零，而且换成向量反而**更慢**（要算 embedding + 向量搜索） | 改口径为「**工具声明占用的上下文 token 数**」与「**首 token 延迟**」。工具数从 200 → 8 时，系统提示词可省数千 token，这才是真收益 |
| 长程连贯性「明显提升」 | 不可证伪 | 定 3 个可判分任务（跨会话指代、历史决策引用、术语一致性），人工/模型打分对比 |

**这是本评估里最需要向上沟通的一条**：指标口径不改，第 3 项的「60%」几乎必然靠自欺达成（比如把 BM25 换成慢路径再"优化"回来）。

### 判断三：依赖红线会卡住第 2 项（Redis + Faiss）

`AGENTS.md` 与安装锁生成器把「依赖变更」当代码评审：

```js
// scripts/generate-coding-agent-install-lock.mjs
["esbuild@0.28.2", "postinstall selects and verifies the platform-specific esbuild binary"],
["protobufjs@7.6.6", "postinstall only warns about protobuf version scheme mismatches"],
// 新增带生命周期脚本的依赖 → errors.push(...) → npm run check 失败
```

现状：**整个允许清单只有 2 条**。而：

- `faiss-node` / `better-sqlite3` 之类原生模块通常带 `install`/`postinstall` 编译脚本；
- `.npmrc` 设了 `min-release-age=2` 与 `save-exact=true`（`README.md` 供应链加固章节）。

结论：第 2 项要么走「外部服务化」（Redis/Faiss 跑在 Node 进程之外，pi 只做 HTTP/gRPC 客户端，零原生依赖），要么先走一次依赖评审加白名单。**前者通常更快也更符合这个仓库的取向。**

---

## 1. 难度矩阵

假设执行人是熟悉本仓库的资深 TS 工程师，1 人。

| # | 需求 | 核心改动面 | 外部依赖 | 风险 | 预计 | 难度 |
|---|---|---|---|---|---|---|
| 1 | ReAct 循环 + 反思 + 重试 | `agent-loop.ts`（核心）、`agent-session.ts`、类型 | 无 | **改核心循环，影响全部消费者** | 8–11 人日 | ★★★☆☆ |
| 2 | Redis + Faiss 分层记忆 | `agent-session.ts`、新包 `pi-memory` | **Redis + 向量库** | 数据模型未定；与既有压缩/会话树冲突 | 25–35 人日 | ★★★★★ |
| 3a | 语义路由 | `tool-search/tool.ts`（现成接口） | embedding 服务 | 低 | 3–5 人日 | ★★☆☆☆ |
| 3b | AST + 子进程沙箱 | `packages/codemode` | 无（Node 内置） | **安全产品，验收难** | 10–15 人日 | ★★★★☆ |
| 4 | WebSocket 全双工 | `protocol/client` + 新网关 | `ws`（纯 JS，无生命周期脚本） | 低 | 6–10 人日 | ★★☆☆☆ |

**合计约 52–76 人日（1 人 11–15 周）**。3 人并行、按依赖排序可压到 6–8 周。

难度排序（从易到难）：**3a 语义路由 < 4 全双工 < 1 循环 < 3b 沙箱 < 2 记忆**。

---

## 2. 推荐实施顺序（按「收益/代价」而非需求编号）

```
第 1 阶段（1–2 周）  3a 语义路由  ── 接口现成，见效快，直接缓解 Context 膨胀
                     1  步数上限  ── 半天工作量，先止血「跑飞」
第 2 阶段（2–3 周）  1  反思 + 重试 ── 依赖第 1 阶段的循环改造
                     4  WS 网关    ── 复用 protocol/server，不新建协议
第 3 阶段（3–4 周）  3b 沙箱      ── 独立子项目，可与 2 并行
第 4 阶段（5–7 周）  2  记忆      ── 必须最后做，且要先缩范围（见下）
```

### 两个「先别做」

1. **别为「当前会话」做记忆。** pi 的会话树 + 投影 + 压缩已经解决了当前会话的上下文管理，而且性质比 Redis 滑动窗口好得多（可分支、可回放、压缩不删原始数据）。第 2 项只应该做**跨会话/跨项目**的召回。范围不收窄，你会在压缩之上再叠一层缓存，两者互相打架。
2. **别用 AST 当安全边界。** AST 只能做「静态预检/提示」，无法阻止 `while(1)`、内存炸弹、原型链逃逸。真安全边界是进程 + 资源限制（或容器）。第 3b 篇会给出正确的分层。

---

## 3. 四项需求的共同「插入点地图」

看懂这张图，四项改动都能定位：

```
                     ┌──────────────────────────────────────────────┐
   输入 ────────────► │ AgentSession.prompt()                        │
                     │   ├─ 斜杠命令 / @file / !shell                │
                     │   ├─ prompt template 展开                     │
                     │   └─ 落盘（SessionManager.append）            │
                     └───────────────┬──────────────────────────────┘
                                     ▼
                     ┌──────────────────────────────────────────────┐
                     │ Agent.prompt() → agentLoop()                 │
                     │                                              │
  【1 步数/反思/重试】│  while(true)          ◄── 这里加计数器        │
                     │   ├─ prepareNextTurn  ◄── 压缩；反射也可挂这  │
                     │   ├─ prepareRequest   ◄── 记忆注入点          │
                     │   ├─ transformContext ◄── 记忆注入点（已链式）│
                     │   ├─ convertToLlm                            │
                     │   ├─ streamFn ────────► pi-ai ──► provider    │
                     │   ├─ 工具批次                                │
  【3 沙箱】          │   │    └─ prepareToolCall ◄── 重试/修复在此    │
  【3 路由】          │   │         └─ execute ─► codemode/沙箱      │
                     │   └─ finishTurn      ◄── 反思门控可挂这       │
                     └───────────────┬──────────────────────────────┘
                                     ▼
                     ┌──────────────────────────────────────────────┐
  【4 全双工】        │ AgentEvent 流 → interactive / print / json   │
                     │              → rpc(JSONL)                    │
                     │              → protocol(CBOR) → client/server │
                     └──────────────────────────────────────────────┘
```

四个插入点的性质差异很大，这是难度差异的根源：

| 插入点 | 稳定性 | 改它的代价 |
|---|---|---|
| `ToolRanker`（接口已留） | 设计时就为替换预留 | **极低** |
| `agent.transformContext` / `prepareRequest` | 已存在且已被 AgentSession 链式包裹 | 低 |
| 传输层（protocol 的 `ByteTransportFactory`） | 抽象良好 | 低 |
| `agent-loop.ts` 的 `runLoop` | 无预留，且是 13 个下游共用的核心 | **高** |
| `codemode` 运行时 | 内部实现，无稳定契约 | 中高 |

---

## 4. 本目录文档

| 文档 | 内容 |
|---|---|
| [01-ReAct循环与自我纠错.md](01-ReAct循环与自我纠错.md) | 步数限制、反思、参数失败重试；二开前后全流程 |
| [02-分层记忆系统.md](02-分层记忆系统.md) | Redis 短期 + 向量长期 + 画像；与既有压缩的关系 |
| [03-工具沙箱与语义路由.md](03-工具沙箱与语义路由.md) | ToolRanker 替换、AST 的定位、子进程隔离 |
| [04-全双工流式与状态同步.md](04-全双工流式与状态同步.md) | WS 传输、COT 透传现状、打断与恢复 |

---

## 5. 一句话回答「难度」

- **不难的部分**：语义路由（接口现成）、WebSocket（传输抽象现成）、步数上限（半天）。
- **中等**：反思与重试（要动核心循环，且必须防「反思自身变成死循环」）。
- **真难**：沙箱（安全产品，验收标准难写）、记忆（数据模型 + 依赖红线 + 与压缩的语义冲突）。

其中**记忆是唯一一项「代码不难但设计很难」**——Redis 和 Faiss 接进去都是几百行，难的是回答「有了会话树和压缩，为什么还需要它、它到底存什么、多久过期、和压缩谁先跑」。
