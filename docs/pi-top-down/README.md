# 自顶向下理解 pi 仓库

> 目标仓库：`2rnight/pi-agent-learning`（pi monorepo，npm 包名前缀 `@earendil-works/*`）
> 本文档集：先给大纲，再逐节点展开。**本目录所有 md 文件合起来就是一套完整的自顶向下导读。**

---

## 0. 一句话概括

Pi 是一个「极简、可扩展的 agent harness（智能体骨架）」：

- 底层把**几十家 LLM 供应商**抽象成统一 API（`pi-ai`）；
- 中间层把「模型 + 工具 + 循环 + 事件」抽象成可复用的 agent 运行时（`pi-agent-core`）；
- 上层落地成一个终端里的编码代理 CLI（`pi-coding-agent`），并用 `pi-tui` 自绘界面、用扩展/技能/提示词模板做「让你自己改造成你的样子」；
- 另有一条相对独立的实验线 `chord`（应用组合运行时、复制状态、RPC），在它之上长出远程会话协议与持久化 agent（`protocol` / `client` / `server` / `durable` / `env`）。

---

## 1. 仓库的物理结构（先看森林）

```
pi-agent-learning/
├── packages/                 # 14 个 workspace 包，是本仓库的全部实质内容
├── docs/                     # 你正在读的导读（本文档集）
├── scripts/                  # 构建、发布、依赖校验、模型数据、性能分析等 40+ 脚本
├── .github/workflows/        # CI、发布、策略门禁、npm 审计
├── .pi/                      # 仓库自我托管：用 pi 开发 pi（prompts/skills/extensions）
├── AGENTS.md                 # 给「人和 agent」的项目规则（也是 pi 读取的项目上下文）
├── tui-plan.md               # TUI 的历史设计文档（36KB）
└── package.json              # 根 workspace + 构建/检查/测试/发布脚本
```

`packages/` 下的 14 个包：

| 包目录 | npm 名 | 角色 | 源码规模（约） |
|---|---|---|---|
| `chord` | `@earendil-works/chord` | 应用组合运行时：facets / services / 复制状态 / RPC | 8.8k 行 |
| `telemetry` | `@earendil-works/pi-telemetry` | 厂商中立的遥测契约 | 0.9k 行 |
| `ai` | `@earendil-works/pi-ai` | 统一多供应商 LLM API | 26.4k 行 |
| `agent` | `@earendil-works/pi-agent-core` | Agent 运行时（循环 + 工具执行 + 事件） | 2.5k 行 |
| `coding-agent` | `@earendil-works/pi-coding-agent` | 编码代理 CLI（产品主体） | 85k 行 |
| `tui` | `@earendil-works/pi-tui` | 差分渲染终端 UI 库 | 19.3k 行 |
| `mcp` | `@earendil-works/pi-mcp` | 独立 MCP 客户端 | 3.2k 行 |
| `codemode` | `@earendil-works/pi-codemode` | QuickJS 沙箱里跑模型写的 JS（唯一能力是调用工具） | 1.7k 行 |
| `protocol` | `@earendil-works/pi-protocol` | CBOR 信封 + 字节流分帧（实验） | 0.9k 行 |
| `client` | `@earendil-works/pi-client` | 传输无关的远程会话客户端 | 1.1k 行 |
| `server` | `@earendil-works/pi-server` | 本地路由服务器（实验） | 2.0k 行 |
| `durable` | `@earendil-works/pi-durable` | 持久化对话/任务/文档运行时 | 19.5k 行 |
| `env` | `@earendil-works/pi-env` | 远程执行环境（Rust daemon + SSH） | 1.6k 行 TS + Rust |
| `evals` | `@earendil-works/pi-evals`（private） | 行为评测（vitest-evals） | 1.4k 行 |

---

## 2. 依赖关系（真正的骨架）

```
                 ┌──────────────┐
                 │  telemetry   │  (无内部依赖)
                 └──────┬───────┘
                        │
                 ┌──────▼───────┐        ┌──────────┐   ┌──────────┐
                 │     ai       │        │   tui    │   │   mcp    │   ┌──────────┐
                 └──────┬───────┘        └────┬─────┘   └────┬─────┘   │ codemode │
                        │                     │              │         └────┬─────┘
                 ┌──────▼───────┐             │              │              │
                 │  agent-core  │             │              │              │
                 └──────┬───────┘             │              │              │
                        └──────────┬──────────┴──────────────┴──────────────┘
                                   ▼
                          ┌─────────────────┐
                          │  coding-agent   │  ← 产品：把上面全部组装起来
                          └─────────────────┘

另一条线（互相独立，除 chord 外自成一簇）：

  chord ──► protocol ──► client
    │           │          ▲
    │           └──► server┘
    └──► durable ──► env
```

**关键判断：这个仓库里其实有两条几乎不相交的产品线。**

1. **Pi 产品线**（`telemetry → ai → agent-core → coding-agent`，外挂 `tui/mcp/codemode`）：成熟、可发布、README 主推。
2. **Chord 实验线**（`chord → {protocol, client, server, durable, env}`）：`chord` 明确声明「不是 Pi 包，可被任何应用使用」；`protocol/client/server/durable` 都标注 experimental。它是「把 agent 从单进程 CLI 变成可远程附着、可持久恢复的服务」的探索。

---

## 3. 大纲（节点清单）

### 第一层：全景

- [01 全景与分层地图](01-全景与分层地图.md) —— 仓库定位、包清单、依赖图、两条产品线怎么读
- [02 一次请求的完整生命周期](02-一次请求的完整生命周期.md) —— 从敲下回车到屏幕刷新，一条线索串起所有层

### 第二层：模型接入层 `pi-ai`

- [03 pi-ai 的三层抽象](03-pi-ai-三层抽象.md) —— `Models` / `Provider` / `Api` 如何解耦，compat 与 lazy 的取舍
- [04 消息与流式事件模型](04-pi-ai-消息与流式事件.md) —— `Message`、内容块、`AssistantMessageEvent` 流、usage 与 stopReason
- [05 Provider 与模型目录](05-pi-ai-Provider与模型目录.md) —— 40+ provider、`models.generated.ts`、模型目录与虚拟模型
- [06 认证与凭证](06-pi-ai-认证与凭证.md) —— API key、OAuth（PKCE/设备码）、credential store、按请求解析

### 第三层：智能体运行时 `pi-agent-core`

- [07 Agent 与状态](07-agent-core-Agent与状态.md) —— `Agent` 类、`AgentState`、队列、订阅、`waitForIdle`
- [08 agent-loop 循环](08-agent-core-agent-loop.md) —— 双层循环、turn、steering / follow-up / finishTurn / prepareRequest
- [09 工具执行管线](09-agent-core-工具执行管线.md) —— prepare → beforeToolCall → execute → afterToolCall → finalize，并行与 terminate
- [10 上下文装配](10-agent-core-上下文装配.md) —— `transformContext` → `convertToLlm`、系统提示词补丁、工具集声明

### 第四层：编码代理应用 `pi-coding-agent`

- [11 分层架构](11-coding-agent-分层架构.md) —— `main.ts` / `cli` / `modes` / `core` / `extensions` 五层职责
- [12 AgentSession](12-coding-agent-AgentSession.md) —— 把 agent-core 变成产品的那个 4300 行类
- [13 会话存储与分支树](13-coding-agent-会话存储与分支.md) —— JSONL v3、`id/parentId` 树、fork/clone/compaction 投影
- [14 内置工具集](14-coding-agent-内置工具集.md) —— read/bash/edit/write/grep/find/ls 的设计约束
- [15 系统提示词与项目上下文](15-coding-agent-系统提示词与项目上下文.md) —— 提示词分节、AGENTS.md、skills、prompt templates
- [16 上下文压缩](16-coding-agent-上下文压缩.md) —— 何时压缩、摘要怎么生成、摘要条目如何替换历史
- [17 扩展机制](17-coding-agent-扩展机制.md) —— jiti 加载、`ExtensionAPI`、事件总线、信任模型
- [18 资源、设置与包管理](18-coding-agent-资源与包管理.md) —— `~/.pi/agent` 布局、settings、Pi packages（npm/git）
- [19 四种运行模式](19-coding-agent-四种运行模式.md) —— interactive / print / JSON / RPC / SDK 的同一内核

### 第五层：支撑库

- [20 pi-tui 终端渲染](20-pi-tui-终端渲染.md) —— 差分渲染、主屏/备用屏、组件与布局
- [21 pi-mcp 与 pi-codemode](21-pi-mcp与pi-codemode.md) —— 外部工具接入的两种方式：协议接入 vs 代码编排
- [22 pi-telemetry](22-pi-telemetry.md) —— 显式上下文传递的遥测契约

### 第六层：Chord 实验体系

- [23 chord 应用组合运行时](23-chord-应用组合运行时.md) —— facets、services、复制状态、delta
- [24 protocol / client / server](24-protocol-client-server.md) —— 4 字节长度 + CBOR 的分帧协议与路由
- [25 durable 与 env](25-pi-durable与pi-env.md) —— 先落盘再展示的 harness，与远程执行环境

### 第七层：工程实践

- [26 构建、测试、CI 与发布](26-工程实践-构建测试与发布.md) —— 构建顺序、check 门禁、供应链加固、发布流水线、evals

---

## 4. 建议阅读路径

- **想理解产品怎么跑起来**：01 → 02 → 07 → 08 → 09 → 11 → 12 → 13 → 19
- **想接模型 / 看懂 provider**：03 → 04 → 05 → 06
- **想做扩展或二次开发**：11 → 12 → 17 → 18 → 19（再回头看 07–10）
- **想改提示词 / 省 token**：10 → 15 → 16
- **想研究分布式 / 远程 agent**：23 → 24 → 25
- **只想跑起来/读代码不迷路**：01 → 02 → 26

## 5. 读这套文档时需要记住的三个「设计轴」

1. **抽象层次**：`Api`（协议实现）↔ `Provider`（厂商封装）↔ `Models`（注册表/门面）；`AgentTool`（运行时）↔ `ToolDefinition`（产品层，带 UI 渲染）；`AgentMessage`（可扩展）↔ `Message`（模型能懂的 4 种）。
2. **状态归属**：谁拥有「真相」？答案几乎总是**转录（transcript）/ 会话文件**。系统提示词、工具集、压缩摘要都是从转录重放出来的，不是独立状态。
3. **可扩展点**：几乎所有行为都被设计成「回调 + 事件」而非硬编码——`streamFn`、`convertToLlm`、`transformContext`、`beforeToolCall`、`finishTurn`、`prepareRequest`、扩展事件。改 Pi 的正确姿势通常是「找到插入点」，而不是改内核。

---

*文档基于 commit `5b6c792`（`fix(ai): retry HTTP/2 pending stream cancellation`）。仓库仍在演进，行号与文件名可能小幅漂移，以 `packages/*/src` 为准。*
