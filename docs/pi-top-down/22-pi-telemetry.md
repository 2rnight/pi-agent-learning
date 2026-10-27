# 22 pi-telemetry：显式传递的遥测契约

`pi-telemetry` 是全仓库最小的包（935 行、6 个文件），但它体现了 pi 的一个明确取舍：**不做全局单例，不绑定后端。**

## 1. 它提供什么，不提供什么

README 列得很清楚：

**提供**：
- 回调式 `TelemetryContext` / `TelemetrySpan` 契约；
- `NOOP_TELEMETRY_CONTEXT`；
- 参考实现 `InMemoryTelemetryContext`；
- 可序列化的 schema 定义 + 类型推导；
- 一致性测试（conformance）。

**不提供**：
- 任何导出器（exporter）；
- 全局 current-span 状态；
- 对任何遥测后端的依赖。

> Applications can use the in-memory reference or provide an adapter for OpenTelemetry, Sentry, logs, or another backend. Pi packages pass telemetry contexts **explicitly** and define their domain schemas separately.

## 2. 核心 API（18 行就能读完）

```ts
export type AttributeValue = string | number | boolean | readonly string[] | readonly number[] | readonly boolean[];

export interface TelemetryContext {
  // 开始一个 span（子 span 的父就是 this）
}

export interface TelemetrySpan extends TelemetryContext {
  // setAttribute / addEvent / setStatus / end
}

export type SpanStatus = { status: "ok" } | { status: "error"; error?: { name: string; message: string } };
```

关键设计：**`TelemetrySpan extends TelemetryContext`**。也就是说「span 本身就是下一个 span 的父上下文」，不需要全局栈、不需要 async context 传播。

```ts
const span = ctx.startSpan("example.account.load");
const child = span.startSpan("example.cache.lookup");
child.end();
span.end({ status: "ok" });
```

**取舍**：显式传参意味着你必须在函数签名里带上 `ctx`（侵入性更强），但换来的是：
- 没有隐式全局状态（多租户/并发安全）；
- 在 worker、VM、远程边界上都不会丢失上下文；
- 测试极其简单（传入 `InMemoryTelemetryContext` 断言）。

对一个**有沙箱（codemode）、远程执行（env）、多进程（server/worker）** 的系统，隐式 async-local 传播会经常断掉，显式传递反而是正确选择。

## 3. 概念模型

README 用一张图解释：

```
example.account.load                        span
├─ attributes: account.id=123, found=true   事实
├─ event: example.cache.lookup              发生的事（无时长）
│  └─ attribute: cache.hit=false
└─ status: ok                               结果
```

| 概念 | 含义 |
|---|---|
| Span | 一次操作的计时记录 |
| Parent/child | span 树，显示时间花在哪 |
| Attribute | 命名的事实（`provider: "openai"`, `cache.hit: true`） |
| Event | span 内的瞬时发生（`retry.scheduled`） |
| Status | `ok` / `error`（含错误名与消息） |
| Context | 新工作挂在树上的位置 |

## 4. 类型化 schema：让遥测字段也可编译期检查

这是这个包最「重」的部分（`index.ts` 后半段全在推导类型）：

```ts
export function defineTelemetrySchema<const T extends TelemetrySchemaDefinition>(schema: T): T;

export type TelemetrySchemaSpanStartAttributes<...>   // 开始 span 时必须给的属性
export type TelemetrySchemaSpanEndAttributes<...>     // 结束时必须给的属性
export type TelemetrySchemaSpanEventName<...>         // 允许的事件名
export type SchemaTelemetrySpan<...>                  // 带类型的 span
export type TypedSpanStarter<Schemas>                 // 带类型的 startSpan
export type ExactTelemetryAttributes<Expected, Actual> // 拒绝多余字段
```

配套的类型工具：

- `TelemetryStartAttributeDefinition & { required: boolean }` —— 区分必填/选填；
- `InferRequiredAndOptionalAttributes` / `InferOptionalAttributes` —— 从定义推导出 TypeScript 类型；
- `TelemetryParentDefinition` —— 声明某个 span 的合法父节点（拓扑约束）。

**为什么值得这么做**：遥测字段是「写入容易、查询痛苦」的典型。拼错 `provider` 为 `providor`，你不会收到任何错误，只会在三个月后发现某个看板数据缺失。**把遥测契约变成类型，是唯一可靠的治理方式。**

## 5. 安全与可移植性

README 有专门一节 "Security and Portability"。要点（结合代码可推断）：

- schema 是**可序列化**的（纯数据），可以跨进程/跨语言传递；
- 属性值限制为 string/number/boolean 及其数组——**不允许任意对象**，避免意外把 prompt、密钥、文件内容塞进遥测；
- 没有导出器 = 数据不会自己跑出去，要不要发、发去哪由应用决定。

在 pi 的语境里这很重要：`pi-ai` 会处理密钥、`coding-agent` 会处理文件内容。遥测包如果自带网络上报，就是一个数据外泄面。

## 6. 它怎么被使用

`pi-ai` 依赖 `pi-telemetry`（依赖图里唯一的下游）。上层把 ctx 一路透传：

```ts
// 概念示意
await models.streamSimple(model, context, { ...options, telemetry });
```

`coding-agent` 里也有 `core/telemetry.ts` 做适配。注意：**`pi-agent-core` 并不依赖 telemetry 包**（依赖图里没有），说明核心循环保持「零遥测依赖」的干净——需要时可从上层注入。

## 7. 6 个文件的分工

```
src/
├── index.ts        # 契约 + 全部类型推导（最大的文件）
├── noop.ts         # NOOP_TELEMETRY_CONTEXT
├── memory.ts       # InMemoryTelemetryContext（参考实现）
├── testing/        # 一致性测试套件（给别人验证自己的 adapter）
└── test/
```

`src/testing/` 的存在说明作者希望**别人实现 adapter**（OpenTelemetry、Sentry、日志），并提供了一个「跑通这套测试就算合格」的验收标准。这是小而完整的库的标志。

## 8. 本章要点

- 显式传递的 span 契约：`TelemetrySpan extends TelemetryContext`，没有全局状态。
- 只给契约 + 内存实现 + 一致性测试，**不提供导出器**（避免数据外泄面）。
- 类型化 schema 把遥测字段变成编译期约束——治理遥测字段的正确姿势。
- 属性值限定为标量/标量数组，schema 可序列化，可跨进程。
- `pi-ai` 依赖它，`pi-agent-core` 不依赖它（核心保持干净）。

下一章：[23 chord 应用组合运行时](23-chord-应用组合运行时.md)
