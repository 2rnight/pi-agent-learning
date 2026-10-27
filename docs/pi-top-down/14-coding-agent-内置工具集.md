# 14 内置工具集

pi 只给模型很少的工具，而且每个工具都受同样的三条约束：**输出要截断、写操作要串行、行为要可替换**。

## 1. 工具清单

`core/tools/` 下的 19 个文件，对应的工具：

| 工具 | 文件 | 说明 |
|---|---|---|
| `read` | `read.ts` | 读文件（支持 offset/limit、图片） |
| `write` | `write.ts` | 写文件 |
| `edit` | `edit.ts` + `edit-diff.ts` | 精确替换 + 生成 diff |
| `bash` | `bash.ts` | 执行命令（有 `BashOperations` 抽象） |
| `powershell` | `powershell.ts` | Windows 对应实现 |
| `grep` | `grep.ts` | 内容搜索 |
| `find` | `find.ts` | 文件名搜索 |
| `ls` | `ls.ts` | 目录列表 |

辅助模块：`truncate.ts`（截断）、`file-mutation-queue.ts`（写串行化）、`path-utils.ts`、`output-accumulator.ts`、`render-utils.ts`、`renderers/`（TUI 渲染）、`tool-definition-wrapper.ts`（`AgentTool` ↔ `ToolDefinition`）。

**注意 `read`/`grep`/`find` 而不是 `cat`/`rg`**：pi 默认提示词里写得很直白——

```ts
export const readToolSystemPromptContribution = {
  snippet: "Read file contents",
  guidelines: ["Use read to examine files instead of cat or sed."],
} as const;
```

原因：工具的输出经过截断、编码处理、图像识别，比裸 shell 命令更可控。

## 2. 约束一：所有输出都要截断

`truncate.ts` 的注释：

> Truncation is based on two independent limits - whichever is hit first wins:
> - Line limit (default: 2000 lines)
> - Byte limit (default: 50KB)
> Never returns partial lines (except bash tail truncation edge case).

```ts
export const DEFAULT_MAX_LINES = 2000;
export const DEFAULT_MAX_BYTES = 50 * 1024;
export const GREP_MAX_LINE_LENGTH = 500;
```

`TruncationResult` 会告诉你截断的全部事实：

```ts
interface TruncationResult {
  content: string;
  truncated: boolean;
  truncatedBy: "lines" | "bytes" | null;
  totalLines: number; totalBytes: number;
  outputLines: number; outputBytes: number;
  lastLinePartial: boolean;
  firstLineExceedsLimit: boolean;
  maxLines: number; maxBytes: number;
}
```

**为什么这些字段要暴露给模型**：截断后的结果里会带「还有 3500 行没显示」这类信息，模型才知道要读下一页（用 `offset`），而不是以为文件就这么长。**这是「工具必须诚实」的具体体现。**

三种截断方向：
- `truncateHead`：保留开头（read 默认）；
- `truncateTail`：保留结尾（bash 输出更适合看尾部）；
- `truncateLine`：单行内截断（grep 命中行，500 字符上限）。

## 3. 约束二：写同一文件必须串行

`file-mutation-queue.ts`：

```ts
/**
 * Serialize file mutation operations targeting the same file.
 * Operations for different files still run in parallel.
 */
export async function withFileMutationQueue<T>(filePath: string, fn: () => Promise<T>): Promise<T>
```

实现细节有意思：

1. **用 `realpath` 做 key**，这样软链接指向同一文件时也会串行（`getMutationQueueKey`）；
2. 文件不存在时（ENOENT/ENOTDIR）退回 `resolve()` 的路径——因为新建文件还没有 realpath；
3. 全局 `registrationQueue` 串行化「注册」这一步，避免竞态创建两个队列。

**为什么需要**：第 09 章说过工具默认并行。模型常见行为是一次发 `edit a.ts` + `edit a.ts`（改两处）。如果真并行，两个「读-改-写」会互相覆盖。所以写类工具内部自己保证 per-file 串行，同时不同文件仍然并行。

## 4. 约束三：行为可替换（这是最有价值的设计）

每个工具都暴露一个 `XOperations` 接口，把「怎么做」和「做什么」分开。以 `read` 为例：

```ts
export interface ReadOperations {
  readFile: (absolutePath: string) => Promise<Buffer>;
  access: (absolutePath: string) => Promise<void>;
  detectImageMimeType?: (absolutePath: string) => Promise<string | null | undefined>;
}

const defaultReadOperations: ReadOperations = {
  readFile: (path) => fsReadFile(path),
  access: (path) => fsAccess(path, constants.R_OK),
  detectImageMimeType: detectSupportedImageMimeTypeFromFile,
};
```

`bash` 更进一步，有 `BashOperations` + `BashSpawnContext` + `BashSpawnHook`。

**这层抽象带来的能力**（仓库里真实用到的）：

- **Gondolin 扩展**：把 read/bash/edit/write 路由进 micro-VM（`docs/containerization.md`）；
- **`pi-env`**：把工具执行送到远程 SSH 机器（第 25 章，`RemoteExecutionEnv` 要求「结果与本地 Node 实现一致，仅错误信息可不同」）；
- **Durable**：在持久化 harness 里用同一套工具语义；
- **测试**：注入假的 operations，不碰真实文件系统。

也就是说：**「工具 = 协议 + 默认实现」**，而不是「工具 = 一段 fs 调用」。这是 pi 能被容器化/远程化的结构性原因。

## 5. `edit` 工具：精确替换与 diff

`edit.ts` + `edit-diff.ts` 的设计：

- 输入是「旧字符串 → 新字符串」（精确匹配），而不是行号或整文件重写——**降低模型幻觉风险**；
- 匹配失败时给出有用的错误（哪个片段没找到、附近有什么）；
- `edit-diff.ts` 负责 `generateDiffString` / `generateUnifiedPatch`，供 UI 显示彩色 diff、供 `pi` 自我审查。

`docs` 里还有 `scripts/edit-tool-stats.mjs`、`read-tool-stats.mjs` 这类脚本——**作者在用真实会话数据统计工具的使用效果**，这是少见的工程实践。

## 6. `bash` 工具与 `!` 命令的区别

两个容易混的东西：

| | `bash` 工具 | `!` 前缀命令 |
|---|---|---|
| 谁发起 | 模型 | 用户 |
| 实现 | `core/tools/bash.ts` | `core/bash-executor.ts`（`executeBashWithOperations`） |
| 结果去哪 | toolResult 消息 → 模型 | `bashExecution` 自定义消息 → 不进模型上下文（默认） |
| 用途 | agent 干活 | 用户手动跑一下，把输出抓进对话 |

两者共享 `BashOperations` 抽象，所以容器化时两者都能被路由（`containerization.md` 里明确提到 "routing built-in tools **and `!` commands**"）。

## 7. `ToolDefinition` 与 `AgentTool`：两个层的工具

这是 `coding-agent` 里一个必须理解的双层结构：

```
AgentTool（agent-core）                ToolDefinition（coding-agent）
─────────────────────                 ────────────────────────────
name/label/description                + renderShell / renderCall / renderResult
parameters (TypeBox)                  + 命名空间、暴露策略（ToolExposure）
execute(...)                          + 触发扩展事件（ToolCallEvent 等）
                                      + 与 UI/权限/会话集成
```

`tool-definition-wrapper.ts` 的 `wrapToolDefinition()` 把后者转成前者。`ExtensionRunner` 里的 `wrapRegisteredTools()` 做同样的事。

`ToolExposure` 值得注意：

```ts
export type ToolExposure = "direct" | "model-only" | "codemode" | "deferred" | "hidden";
```

- `direct`：直接出现在模型工具列表里；
- `codemode`：只通过代码模式调用（不进上下文，省 token）；
- `deferred`：按需加载（配合 `tool_search`）；
- `hidden`：只能通过命令/扩展调用。

**这是「工具太多撑爆上下文」的解法**：工具可以存在但不出现在模型眼前（第 17、21 章）。

## 8. 图像与二进制

`read` 遇到图片时：

- 检测 MIME（`detectSupportedImageMimeTypeFromFile`）；
- `processImage()` 缩放/转换（`utils/image-process.ts`、`image-convert.ts`、`image-resize.ts`）；
- 变成 `ImageContent` 块进消息。

`normalizeToolResultImages()` 在 `afterToolCall` 阶段统一处理图片，避免每个工具各写一遍。

## 9. 本章要点

- 只有 8 个工具，但每个都遵守三条约束：**截断、写串行、可替换**。
- 截断是双阈值（2000 行 / 50KB），并且把「被截断了」的事实告诉模型。
- `withFileMutationQueue` 用 realpath 做 key，保证同一文件的「读-改-写」不互相覆盖。
- `XOperations` 抽象让工具能跑在容器、远程机器、假实现上——这是容器化与 durable 的基础。
- `ToolDefinition`（产品层）与 `AgentTool`（运行时层）之间靠 wrapper 转换，`ToolExposure` 控制工具是否进模型上下文。

下一章：[15 系统提示词与项目上下文](15-coding-agent-系统提示词与项目上下文.md)
