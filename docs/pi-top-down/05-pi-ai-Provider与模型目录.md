# 05 Provider 与模型目录

这一章讲清楚：**pi 怎么知道「世界上有哪些模型」，以及怎么把「用户选的模型」变成一个能发请求的对象。**

## 1. 三个层次的模型数据

```
① 生成的静态数据      models.generated.ts   ← scripts/generate-models.ts（来自 models.dev / openrouter 等）
② 运行时远端目录      ModelCatalog           ← 从 pi.dev 拉，带 ETag / Last-Modified
③ 本地配置/虚拟模型   models.json / virtual-models.ts
```

### ① 静态生成：`models.generated.ts`

`AGENTS.md` 明确禁止手改：

> Never modify `packages/ai/src/models.generated.ts` directly; update `packages/ai/scripts/generate-models.ts` instead, then regenerate.

生成器 `scripts/generate-models.ts` 干的事：读 `models-dev-reasoning-options.ts`、`openrouter-catalog.ts` 等数据源，产出每个 provider 的 `*.models.ts` 和聚合的 `models.generated.ts`，并提供校验：

- `validateModelDataDirectory()` / `validateGeneratedModelData()`；
- `assertExactModelIds()` —— 模型 id 必须精确匹配，防止上游改名字悄悄漏掉；
- `createModelDataManifest()` —— 记录数据版本。

CI 里 `npm run check:model-data` 会验证「生成脚本 + 数据」是否与提交的文件一致。

**为什么这么设计**：模型元数据（上下文窗口、价格、thinking 等级映射）是外部事实。硬编码在代码里必然过期；手改生成文件则不可复现。所以：数据是数据，代码是代码，中间用生成器隔开。

### ② 运行时刷新：`model-catalog.ts` + `models-store.ts`

```ts
// packages/ai/src/models-store.ts
export interface ModelsStoreEntry {
  models: readonly AnyModel[];
  lastModified?: number;  // 远端 Last-Modified
  checkedAt?: number;
  etag?: string;          // 原样保存（含引号），下次作为 If-None-Match 回传
}
export interface ModelsStore {
  read(providerId): Promise<ModelsStoreEntry | undefined>;
  write(providerId, entry): Promise<void>;
  delete(providerId): Promise<void>;
}
```

这是标准的 **HTTP 条件请求缓存**模式：内置数据是基线，远端目录更新则覆盖，同时把 ETag 存下来减少流量。

`nix/model-catalog.json` 出现在仓库里，正是因为 **Nix 构建必须离线**：把远端目录的某个 revision 钉死在文件里，运行时再照常覆盖。

### ③ 虚拟模型（virtual models）

`packages/coding-agent/src/core/virtual-models.ts` 的开头注释直接给出定义：

> Virtual models are catalog entries that route each request to a physical model. A virtual model never reaches a provider.

意思是：用户可以在模型列表里看到一个叫「smart」的条目，但它其实不是真模型——它是个**路由器**。每次请求按 `ModelRouteReason` 决定实际发给哪个物理模型：

```ts
export type ModelRouteReason = "user" | "continuation" | "retry" | "direct";
```

- `user`：用户刚发消息后的第一个请求；
- `continuation`：工具结果之后、扩展消息之后的请求；
- `retry`：自动重试（含上下文溢出后的压缩重试）；
- `direct`：agent 循环之外的请求（压缩摘要、扩展调用）。

**用例**：便宜的模型跑连续工具调用，贵的模型只回答用户的第一个问题。这种「按请求类型路由」在 pi 里是**一等公民**，不需要写扩展。

虚拟模型的状态存在会话里（`pi.virtual-model-state` 自定义条目），所以 resume 后路由决策可复现。

## 2. Provider 工厂长什么样

以 `providers/anthropic.ts` 为例（同类结构在 40+ 文件里重复）：

```
providers/anthropic.ts          → 导出 anthropicProvider(): Provider
providers/anthropic.models.ts   → 该 provider 的模型清单数据
```

Provider 需要提供：

| 能力 | 说明 |
|---|---|
| `id` | 唯一标识，模型条目里引用它 |
| 认证描述 | API key / OAuth / 环境变量名 |
| base URL / headers | 含 GitLab Duo、Copilot、Cloudflare Gateway 等特殊头 |
| 模型清单 | 静态或从远端 refresh |
| `streams` | `{ stream, images?, classifier? }` |

`providers/all.ts` 是「一键注册所有 provider」的聚合入口，`providers/faux.ts` 是测试用的假 provider（`coding-agent/test/suite/` 全靠它，见第 26 章）。

## 3. 一个 provider 可以「没有自己的协议实现」

这是最容易被误解的一点。看 `providers/` 里的这些名字：Groq、Cerebras、Together、DeepSeek、Fireworks、Baseten、Nebius… 它们大多**不写新协议**，只是把 `openai-completions` 配一个 base URL 和数据。

真正的协议差异集中在 `api/` 里的 20 多个文件。而 `openai-completions.ts` 内部有一个极其详细的 `OpenAICompletionsCompat`（`types.ts` 里有 30+ 个开关）：

```ts
export interface OpenAICompletionsCompat {
  supportsStore?: boolean;
  supportsDeveloperRole?: boolean;
  supportsReasoningEffort?: boolean;
  maxTokensField?: "max_completion_tokens" | "max_tokens";
  requiresToolResultName?: boolean;
  requiresAssistantAfterToolResult?: boolean;
  requiresThinkingAsText?: boolean;
  thinkingFormat?: "openai" | "openrouter" | "deepseek" | "together" | "zai"
                 | "qwen" | "chat-template" | "qwen-chat-template"
                 | "string-thinking" | "ant-ling";
  // ...
}
```

**这就是「兼容 OpenAI」的真实成本**：每家的 thinking 参数格式、max_tokens 字段名、工具结果格式都不一样。这些开关默认从 URL 自动推断（注释里反复写 `Default: auto-detected from URL`），也可以用 `models.json` 显式覆盖。

## 4. 模型对象里有什么

`Model<TApi>`（`types.ts`）是请求的核心输入，关键字段：

| 字段 | 作用 |
|---|---|
| `id` / `provider` / `api` | 路由三要素 |
| `contextWindow` / `maxOutputTokens` | 上下文管理、截断判断 |
| `cost` | 输入/输出/缓存价格，用于 `calculateCost()` |
| `thinking` 相关（`ThinkingLevelMap` 等） | 哪些档位可用、映射成供应商的什么值 |
| `promptCache` | 缓存保留策略（`CacheRetention: "none"\|"short"\|"long"`） |
| `compat` | 上面的方言开关 |

`models.ts` 里三个工具函数几乎无处不在：

```ts
calculateCost(model, usage)              // 算钱
getSupportedThinkingLevels(model)        // 这个模型支持哪些档位
clampThinkingLevel(model, level)         // 把用户请求的档位夹到支持的范围内
modelsAreEqual(a, b)                     // 模型切换检测
```

`clampThinkingLevel` 特别值得注意：用户在全局设置里写了 `high`，但切到一个只支持 `off/medium` 的模型时，pi 不会报错，而是**静默夹取**——因为「思考档位」是偏好不是契约。

## 5. 请求链路：选模型 → 发请求

```
用户/CLI: --provider anthropic --model claude-sonnet-4-6
   │
   ▼
coding-agent/model-resolver.ts   解析 + 校验 + 覆盖（CLI > 会话 > 设置 > 默认）
   │
   ▼
coding-agent/model-registry.ts   合并静态目录 + 远端目录 + models.json + 虚拟模型
   │
   ▼
ModelRuntime   →  Agent.state.model: Model<Api>
   │
   ▼
Models.streamSimple(model, ctx, opts)
   │  查 providerMap[model.provider].streams.stream
   │  查 apiImplMap[model.api]
   ▼
api/openai-completions.ts  →  fetch / SSE  →  AssistantMessageEventStream
```

`model-resolver.ts` 有 783 行、`model-runtime.ts` 有 1032 行——**「选一个模型」这件事比想象中复杂**，因为要处理：CLI 参数、会话内切换、设置里的 scope、provider 别名、旧模型名迁移、虚拟模型、以及「模型不可用时给什么提示」。

## 6. 当模型不可用时的设计

`auth-guidance.ts` 里有两个格式化函数专门做这件事：

```
formatNoApiKeyFoundMessage()
formatNoModelSelectedMessage()
```

它们属于「错误即引导」的思路：不是抛一个 500，而是告诉用户「你现在可以 `/login`、或设置某个环境变量、或 `--model`」。对一个面向终端的工具，这比错误码重要得多。

## 7. 本章要点

- 模型数据分三层：**生成（可复现）→ 远端刷新（条件请求）→ 本地/虚拟（用户态）**。
- Provider 复用 Api 方言；差异靠 `compat` 开关吸收，而不是复制协议实现。
- **虚拟模型 = 请求路由器**，按 `user/continuation/retry/direct` 选物理模型，状态随会话持久化。
- 模型选择是一条长链路（resolver → registry → runtime），失败时给引导而不是错误码。

下一章：[06 认证与凭证](06-pi-ai-认证与凭证.md)
