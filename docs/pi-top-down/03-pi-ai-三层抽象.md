# 03 pi-ai 的三层抽象

`pi-ai` 解决一个非常具体的问题：**40 多家供应商、20 多种 API 方言、上百个模型，怎么让上层只写一行 `streamSimple(model, ctx, opts)`。**

## 1. 先看命名：Api / Provider / Model 不是一回事

```
Model      = 一个具体模型条目（id、provider、api、上下文窗口、价格、思考能力…）
Provider   = 一个厂商/端点的封装（认证方式、base URL、可选模型、stream 实现）
Api        = 一种「线上协议方言」（openai-completions、anthropic-messages、google-generative-ai…）
```

关系：**一个 Api 实现被多个 Provider 复用**。比如 `openai-completions` 这一种方言，同时支撑 OpenAI、Groq、Cerebras、DeepSeek、Together、xAI、本地 llama.cpp…… 这是整个包能覆盖 40+ 厂家的根本原因：**按方言复用，而不是按厂商复制**。

`packages/ai/src/types.ts`：

```ts
export type KnownApi =
  | "openai-completions" | "openai-responses" | "anthropic-messages"
  | "google-generative-ai" | "google-vertex" | "bedrock-converse-stream"
  | "mistral-conversations" | ... ;
export type KnownProvider = "openai" | "anthropic" | "google" | "groq" | ...;
```

注意 `Api = KnownApi | (string & {})`：**已知的做类型补全，未知的也允许**（自定义端点、企业内网模型）。

## 2. 三层各自的职责

```
┌─────────────────────────────────────────────────────────┐
│ Models（注册表 / 门面）                                  │
│  setProvider() / getModel() / streamSimple() / refresh() │
└───────────────┬─────────────────────────────────────────┘
                │ 查表：model.api → 找到对应 api 实现
┌───────────────▼─────────────────────────────────────────┐
│ Provider（厂商封装）                                     │
│  id、认证（apiKey/OAuth）、baseUrl、模型清单、headers    │
│  返回 ProviderStreams { stream, images?, classifier? }   │
└───────────────┬─────────────────────────────────────────┘
┌───────────────▼─────────────────────────────────────────┐
│ Api（协议实现）                                          │
│  stream(model, context, options) → AssistantMessageEventStream │
│  负责：请求体构造、SSE/WS 解析、增量转成统一事件         │
└─────────────────────────────────────────────────────────┘
```

### `Models` 接口（`packages/ai/src/models.ts`，1261 行）

关键方法与能力：

- `setProvider(provider)` / `getProvider(id)`；
- `getModel(provider, modelId)`、模型列表；
- `streamSimple(model, context, options)` —— **上层唯一需要调的流式入口**；
- `stream(model, context, options)` —— 带完整选项的版本；
- `complete` / `completeSimple` —— 一次性拿完整消息（压缩摘要、分类都用它）；
- `refresh()` —— 从远端模型目录刷新（`ModelsRefreshOptions`）；
- `images` / `classifier` —— 图像生成与分类模型也走同一套注册表。

`createModels()` 是工厂（`models.ts:990`），`createProvider()` 是 Provider 工厂（`models.ts:1039`）。

### `Provider` 的核心形状

```ts
export interface Provider<TApi extends Api = Api> {
  id: ProviderId;
  // 认证、baseUrl、headers、模型清单…
  streams: ProviderStreams;   // { stream, images?, classifier? }
}
```

Provider 是**声明式 + 少量函数**：它告诉注册表「我叫什么、我能提供哪些模型、我怎么发请求」，而不是自己实现协议。

### `Api` 的契约（最重要的类型）

```ts
// packages/ai/src/types.ts
export interface ProviderStreams {
  stream: StreamFunction;          // (model, context, options) => AssistantMessageEventStream
  images?: ImagesFunction;
  classifier?: ClassifierFunction;
}
```

**契约里最反直觉的一条**：`StreamFunction` 不能抛异常。所有错误（网络失败、限流、认证过期、模型拒答）都要编码成流里的 `error` 事件，最终产出一条 `stopReason: "error" | "aborted"` 的 `AssistantMessage`。

为什么？因为 agent loop 需要一条完整的事件序列来维持「每个开始都有结束」的不变式；抛异常会让 UI 卡在半截消息上。

## 3. 目录结构就是架构

```
packages/ai/src/
├── types.ts            # 全部类型：Api / Model / Message / StreamOptions / Event
├── models.ts           # Models 注册表 + createModels/createProvider
├── models.generated.ts # 生成的模型清单（禁止手改）
├── model-catalog.ts    # 模型目录的类型推导（按 provider 分组）
├── api/                # 20+ 协议实现，每个都配一个 .lazy.ts
│   ├── anthropic-messages.ts + .lazy.ts
│   ├── openai-completions.ts + .lazy.ts
│   ├── openai-responses.ts …
│   └── transform-messages.ts   # provider 无关的消息格式转换
├── providers/          # 40+ 厂商工厂，每个配 *.models.ts
│   ├── openai.ts /  openai.models.ts
│   ├── anthropic.ts / anthropic.models.ts
│   └── all.ts          # 一键注册
├── auth/               # 凭证与 OAuth
│   ├── credential-store.ts / resolve.ts / context.ts
│   └── oauth/{anthropic,github-copilot,openai-codex,xai,openrouter,…}.ts
├── compat.ts           # 旧的「全局 API」兼容层
└── utils/              # event-stream / retry / overflow / text / uuid
```

**注意每个 api 文件都有 `.lazy.ts` 兄弟**——这是打包体积控制的关键，见下节。

## 4. compat 与 lazy：一个成熟包的包袱与解法

### `compat.ts`（302 行）

早期 pi-ai 可能是一个「全局单例 + 顶层函数」的包（`stream(model,...)`）。现在改成注册表（`Models`）后，旧的调用方式仍然被大量代码依赖（`coding-agent` 里到处是 `import { streamSimple } from "@earendil-works/pi-ai/compat"`）。

`compat.ts` 就是这个兼容层：把「全局 API」保持可用，内部委托给默认注册表。`legacy-api-aliases.ts` 再补上旧名字映射。

**取舍**：`index.ts` 明确写着「Core only, side-effect free: no generated catalogs, no provider factories, no api-registry, no OAuth implementations, no compat」。也就是说——**新代码请从子路径导入**（`/providers/*`、`/api/*`），`compat` 只是历史债。

### `.lazy.ts` 模式

每个 `api/x.ts` 都有 `api/x.lazy.ts`。懒加载的意义：

- **浏览器/边缘运行时**只想用一家 provider，不该打包 20 种协议；
- **CLI 冷启动**不该解析 26k 行代码（pi 对启动时间很敏感，`core/timings.ts` 专门在测）；
- `.lazy.ts` 提供「先占位、第一次调用才 import 真实现」的薄壳。

对应地，`package.json` 里 `sideEffects` 只列出 `compat.js`、`images.js`、`providers/images/register-builtins.js` —— 明确告诉打包器「其余都可摇树」。

## 5. 模型清单为什么是「生成的」

`models.generated.ts` 309 行，但 `scripts/generate-models.ts` 是数据源。`AGENTS.md` 有硬性规则：

> Never modify `packages/ai/src/models.generated.ts` directly; update `packages/ai/scripts/generate-models.ts` instead, then regenerate.

原因：模型元数据（上下文窗口、价格、是否支持 thinking、支持的 thinking 等级映射）是**外部事实**，必须可复现、可 diff、可被 CI 检查（`npm run check:model-data`）。手改会漂移。

此外还有运行时刷新：`model-catalog.ts` + `models-store.ts` 支持从 pi.dev 拉「更新版本的目录」覆盖内置数据（Nix 构建时用 `nix/model-catalog.json` 钉住的离线快照）。第 05 章展开。

## 6. 三层抽象带来的实际好处

| 场景 | 需要改什么 |
|---|---|
| 新增一家兼容 OpenAI 的厂商 | 加一个 `providers/xxx.ts`（复用 `openai-completions` api），零协议代码 |
| 支持一种全新协议 | 加一个 `api/xxx.ts` + `.lazy.ts`，所有 provider 都能复用 |
| 用户想接公司内网端点 | `models.json` 配置 → 走「自定义 provider」路径，不碰源码 |
| 上层想统一记录所有请求 | 用 `Models` 的 request transforms / `onPayload` / `onResponse` 钩子 |

## 7. 本章要点

- **Api 是方言，Provider 是厂商，Models 是注册表**；复用发生在 Api 层，所以能覆盖 40+ 厂商。
- `StreamFunction` 不许抛异常，错误必须变成 `stopReason: "error"` 的消息——这是 agent 循环不变式的前提。
- `index.ts` 是「无副作用核心」，`compat.ts` 是历史兼容，`*.lazy.ts` 是打包/冷启动优化。
- 模型数据是生成的，不是手写的。

下一章：[04 消息与流式事件模型](04-pi-ai-消息与流式事件.md)
