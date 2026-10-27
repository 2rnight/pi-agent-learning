# 25 durable 与 env：持久化 agent 与远程执行

这两个包是「agent 作为服务」的地基：

- **`pi-durable`**：先落盘、再展示；进程死了能接着跑。
- **`pi-env`**：把工具执行搬到另一台机器（通常通过 SSH），而 worker、存储、凭证留在本地。

## 1. durable 的核心承诺

> Conversations, model turns, tool calls, and your own state are **committed to storage before anything is shown**. If the process dies mid-turn, reopening the storage picks the work up where it stopped.

这是一个非常强的语义：**崩溃一致性**。对比 `coding-agent`（会话是「尽力落盘」，崩溃可能丢最后几条），durable 是「先落盘再可见」。

## 2. 概念模型（这部分是全包最重要的）

| 概念 | 含义 |
|---|---|
| **Harness** | 一份打开的存储 + 在上面跑 agent 的全部机械。**所有变更走一条原子提交线** |
| **Conversation** | 一段转录。`root()` 首次创建；可以 fork。句柄无状态，按 `id` 比较 |
| **Entry** | 不可变转录记录：`pi.user` / `pi.assistant` / `pi.tool-result` / `pi.system` / `pi.reset` / 自定义 |
| **Commit** | 原子写入。一次可以「追加 entry + 改文档 + 建任务」，要么全成要么全不成 |
| **Document** | 与转录并列的类型化 JSON 状态，在 commit 里修改 |
| **Task** | 每步存检查点的持久状态机；重启后从最后一个检查点继续 |
| **Submission** | 交给 conversation 的东西（用户输入或待写 entry），可等待 |
| **Turn / Run** | turn = 一次模型响应 + 它的工具调用；run = 从输入到最终回答的所有 turn |
| **Extension** | 命名的 bundle：工具、提示词分节、钩子、包装器、任务 |
| **Registry** | 本进程安装的扩展集合；运行中可变，新工作用新状态 |
| **Agent** | conversation 用什么跑：模型、思考档位、扩展、工具、指令、工作目录（存在 `pi.agent` 文档里） |

### 内置文档

| 文档 | 存什么 |
|---|---|
| `pi.agent` | agent 选择（模型、扩展、工具、指令、cwd），存名字，使用时才解析 |
| `pi.provider` | 面向供应商的会话身份（缓存亲和） |
| `pi.live` | 正在跑的 generation 与 tools |
| `pi.inbox` | 排队的 submission |
| `pi.usage` | 花费 |

### 一次回答的完整结构

```
submit(input) → pi.user
  pi.generation → pi.system（仅当提示词或工具变化）, pi.assistant（含工具调用）
    pi.tool × n → pi.tool-result × n    ← 由 generation 拥有并等待
  pi.generation → pi.assistant（最终回答） → submission 完成
```

**注意 `pi.system`「仅当提示词或工具变化」时才写**：这和 `pi-agent-core` 的「工具集声明差分」思路一致——避免转录膨胀。

## 3. 任务图：durable 的关键抽象

`src/harness/` 下有一整套：

```
harness.ts        # Harness.open()，提交线
generation.ts     # GenerationTask：调模型，拥有 pi.tool 任务，等它们，再交给下一个 generation
tool.ts           # ToolTask
compaction.ts     # CompactionTask + CompactionCheckpoint
scheduler.ts      # 任务调度
task-graph.ts     # 父子/依赖关系
inbox.ts / submissions.ts  # 队列与等待
live.ts           # 进行中的状态
events.ts         # watchEvents()：观察流
```

几个值得注意的导出：

```ts
export { ConversationBusy, ReadAfterWrite, StorageRejected } from "./errors.ts";
export { defineExtension, defineTool, hook, section, wrapSection, wrapTool } from "./harness/define.ts";
```

- **`ConversationBusy`**：一个 conversation 同时只能有一个 run（显式错误而不是阻塞）。
- **`ReadAfterWrite`**：读到了刚写但还没稳定的状态——这是一致性模型的显式信号。
- **`defineExtension` / `hook` / `section` / `wrapSection` / `wrapTool`**：声明式扩展，对应概念表里的 Extension。

## 4. 存储后端

```
src/storage/
├── memory.ts     # 测试/演示
├── jsonl/        # 文件
└── sqlite/       # 生产
```

`Harness.open(storage, services, context)` —— **存储是注入的**，这和 `coding-agent` 的 `SessionManager` 不同（那个固定 JSONL）。

`src/testing/` 提供一致性测试，保证三个后端语义一致。这和第 22 章 telemetry 的做法一样：**契约 + 参考实现 + 一致性测试**。

## 5. durable 与 agent-core 的关系

依赖图：

```
durable ──► ai
      └──► chord
```

**它不依赖 `pi-agent-core`。** 这是刻意的：

- `agent-core` 的循环是「内存里跑一遍」，事件发出去就完了；
- `durable` 的循环是「每一步都是持久状态机的一个转移」，必须能从中断点恢复。

所以 durable 在 `harness/agent.ts` 里自己实现了 generation 的调度，只复用 `pi-ai` 的流与类型，以及 chord 的复制状态/context。

`DEFAULT_COMPACTION_POLICY`、`DEFAULT_PROGRESS_POLICY`、`DEFAULT_RETRY_POLICY` 是三个内置策略——**策略是数据，不是硬编码**。

## 6. durable 的已知限制（README 原文标注）

- 整个包标 **Experimental. The API changes without notice between releases.**
- 有 `src/env/` 目录，说明它有自己的 `ExecutionEnv` 抽象（文件操作、命令执行、watch），`NodeExecutionEnv` 是本地实现。

这个 `ExecutionEnv` 就是 `pi-env` 插入的地方。

## 7. pi-env：把工具执行搬到远端

> an agent's tools run on another machine, usually over SSH, while the Durable worker, its storage and credentials stay local.

两个组成部分：

1. **`daemon/`（Rust 程序）**：跑在远端，从 stdin 读帧、向 stdout 写帧，执行文件操作与命令；
2. **`RemoteExecutionEnv`（TS）**：实现 Durable 的 `ExecutionEnv`，通过 `Connection` 与 daemon 通信。

```ts
const target = { host: "gpu-box", knownHostsFile: "/data/ssh/known_hosts", hostKeyAlias: "pi-env-gpu" };

// 一次性：让所有者核对指纹
const { lines, fingerprints } = await scanHostKey(target);
await acceptHostKey(target, lines);

// 检测远端系统、按需部署本版本 daemon（SHA-256 校验后才运行）、返回 SSH 连接
const { connection } = await connectSsh(target);
const env = new RemoteExecutionEnv({ connection, id: "pi-env:gpu", cwd: "/home/me/project" });
```

### 安全细节（值得逐条看）

- **主机密钥**：用应用自己的 known-hosts 文件 + 固定 alias；首次使用必须由所有者**带外核对指纹**后 `acceptHostKey`。
- **SSH 参数**：`BatchMode`（不交互）、严格主机密钥检查、禁用转发、不转发本地 locale。
- **daemon 部署**：版本不匹配才部署，**校验 SHA-256 后才执行**。
- **Windows**：检测与部署走 PowerShell。

### 线协议（`docs/protocol.md`）

```
每个帧：
u32 length      bytes after this field
u8  type
u32 id          request id；0 表示不属于任何请求
u32 jsonLength
json            UTF-8 JSON 对象
bytes           原始 payload（剩余长度）
```

所有整数大端，单帧最大 16 MiB（更大的 payload 由操作自己分片）。

帧类型：

| type | 名称 | 方向 | 含义 |
|---|---|---|---|
| 1 | request | client → daemon | `json.op` 指定操作 |
| 2 | result | daemon → client | 成功，请求结束 |
| 3 | error | daemon → client | 失败，请求结束 |
| 4 | event | daemon → client | 进行中请求的进度（exec 输出） |
| 5 | cancel | client → daemon | 取消；仍会以 result/error 结束 |
| 6 | ping | 双向 | 存活探测 |

**「同步行」的设计很实用**：远端 shell 启动文件可能往 stdout 打印东西。daemon 的第一行输出是 `PI-ENV <token>\n`，客户端丢弃它之前的一切。这解决了一个真实世界里非常烦人的问题。

**错误语义**：`{ code, message, syscall?, path? }`，`code` 是 Node 风格（`ENOENT`、`EISDIR`）或执行码（`shell_unavailable`、`spawn_error`、`timeout`、`aborted`、`unknown`）。**只有 code 是契约的一部分**，message 只是诊断。

**路径**：line 协议上是绝对路径字符串；客户端在发送前解析相对路径、`~`、`file://` URL。**路径解析留在客户端**，让远端实现简单。

### 等价性要求

> Its results match `NodeExecutionEnv` running on the remote machine; only error messages may differ.

这是「远端执行」最难的部分：**同一份工具代码在本地和远端必须产生相同结果**。仓库用两种方式验证：

- Durable 的 env 一致性测试套件跑在 `RemoteExecutionEnv` 上；
- 对同一台机器上的 `NodeExecutionEnv` 做**随机操作序列对比**。

### 支持与不支持

支持：Linux、macOS、Android(Termux)、Windows，x86-64 与 arm64。Windows 上字符串命令走 Git Bash（与 `NodeExecutionEnv` 一致），argv 命令直接执行。

不支持（README 明确）：输出窗口化（所有输出都传输）、daemon 侧文件监听（客户端轮询）。

## 8. 把 23/24/25 串起来

```
浏览器/终端客户端
   │  pi-protocol（CBOR 帧）
   ▼
pi-server（不透明路由 + 附着管理）
   │  chord 服务调用
   ▼
Session worker 进程
   │  pi-durable（先落盘再展示，任务检查点）
   ├── pi-ai（调模型）
   └── ExecutionEnv ──► pi-env daemon（SSH 到 GPU 机器跑工具）
```

这条链路解释了为什么 `chord` 的 context（可跨进程取消）、`durable` 的提交语义、`env` 的 daemon 协议都必须存在——**它们各自负责崩溃/断线/跨机器中的一个维度**。

## 9. 本章要点

- durable = **先提交再可见**；核心抽象是 Harness / Conversation / Commit / Document / Task / Submission。
- 任务图把「调模型」「调工具」「压缩」都变成可恢复的状态机，每步存检查点。
- durable 不依赖 agent-core，自己实现调度（因为需要中断恢复）。
- env = Rust daemon + TS 客户端；帧协议有 token 同步行、错误码契约、16 MiB 上限。
- 远端执行的目标是「与本地实现行为等价，只有错误消息可不同」，并用一致性测试 + 随机对比验证。

下一章：[26 构建、测试、CI 与发布](26-工程实践-构建测试与发布.md)
