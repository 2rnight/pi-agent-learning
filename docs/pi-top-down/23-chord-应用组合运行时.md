# 23 chord：应用组合运行时

> Chord is an application-composition runtime for systems assembled from plugins/extensions.
> It is developed as a standalone package in the Pi monorepo, but **it is not a Pi package**: it does not depend on any other Pi workspace package and can be used by unrelated applications.

这是整个仓库里最「自成一体」的包。理解它的动机，比记住 API 更重要。

## 1. 它要解决的问题

> A single application feature may need to run in several environments: for example, an agent worker, a terminal UI, and a remote WebUI.

一个功能（比如「会话列表」）需要同时跑在：

- 后端 worker（有数据库、有凭证）；
- 终端 UI（只有显示能力）；
- 远程 WebUI（隔着网络）；

如果为每个环境写一遍，会得到三份不一致的逻辑。Chord 的答案是：

```
一个插件 → 拆成若干 facet → 每个 facet 运行在它该在的进程/环境
         它们之间通过 service 通信，通过 replicated state 同步
```

## 2. 六个核心概念

| 概念 | 一句话 | 代码 |
|---|---|---|
| **Plugin / Facet** | 同步的装配单元，声明「我提供什么服务、需要什么服务」 | `defineFacet`, `createFacetHost` |
| **Service** | 类型化、稳定的 token；singleton 或 keyed | `defineService` |
| **Replicated state** | 权威状态 + 原子 overlay 事务，消费者拿到不可变完整值 | `replicatedState`, `change(ctx, cb)` |
| **Delta** | 记录/合并 JSON 操作，产生可收敛的批次 | `@earendil-works/chord/delta` |
| **Remote service source** | 把「本地没有的服务」通过适配器接到远端 | `createRemoteServiceBinding`, `RemoteServiceProvider` |
| **Context** | 类 Go 的 context：取消 + 调用作用域内的值 | `@earendil-works/chord/context` |

## 3. Facet：比「模块」更细的部署单元

```
Plugin
 ├─ facet: backend    → 跑在服务端进程（有文件和数据库权限）
 ├─ facet: tui        → 跑在终端进程（只有渲染）
 └─ facet: web        → 跑在浏览器（只有 fetch/WebSocket）
```

装配流程（README）：

1. 所有 plugin **同步**声明自己的形状（提供/依赖哪些服务）；
2. host 校验完整依赖图（缺依赖 = 启动失败，不是运行时才发现）；
3. 绑定服务；
4. **先激活 provider，再激活 consumer**；
5. 按依赖的逆序释放资源。

`FacetKernel`（`facets/host.ts`）就是这个调度器。`facets/loader.ts` 负责加载，`node/` 下有 `bundle.ts` / `bundle-loader.ts` / `manifest.ts` / `package.ts` ——**facet 是被单独打包的**，这样它才能被投递到不同进程。

**这套设计和 pi 的扩展机制的差别**：

| | pi 扩展（第 17 章） | chord facet |
|---|---|---|
| 粒度 | 一个 TS 模块 | 一个插件的一个部署面 |
| 运行位置 | 同一个进程 | 任意进程/环境 |
| 通信 | 直接函数调用 | service / replicated state / 远端适配器 |
| 目标 | 定制一个 CLI | 组合一个分布式应用 |

## 4. Service：稳定门面 + 可替换 provider

```ts
defineService(...)   // 类型化 token
```

两种形态：

- **singleton**：一个 provider；
- **keyed**：动态的按 key 实例化。

关键承诺（README）：

> Consumers retain a stable facade while a provider disconnects or is replaced.

也就是说：provider 崩溃/被替换，consumer **不重新绑定**，只是暂时拿到「不可用」。这在跨进程场景里是必须的——网络抖动不该让整个应用重建对象图。

相关文件：

```
services/provider.ts     # RemoteServiceProvider, createRemoteServiceEndpoint
services/consumer.ts     # 消费端门面
services/handle.ts       # 绑定句柄
services/instances.ts    # keyed 实例管理
services/loopback.ts     # 同进程直连（优化路径）
services/state.ts        # 状态订阅
services/wire.ts         # 线上格式
services/state-codec.ts  # 状态编解码（每个客户端独立 path codec）
services/errors.ts       # RemoteServiceError + 错误码
```

## 5. 复制状态：用「不可变 + 操作批次」代替锁

README 的描述很精确：

> Producers publish atomic overlay transactions with `change(context, callback)`; consumers receive complete immutable values. Draft proxies exist only during the callback and become unusable afterward.

流程：

```ts
const state = replicatedState(initial);

state.change(ctx, (draft) => {
  draft.output += "done\n";     // draft 只在回调期间可变
  draft.entries.push({ id: 1 });
});                              // 回调结束 → draft 句柄失效
```

内部发生的事：

1. **Preparation**：把 draft 变更物化成一个**结构共享的不可变候选值** + **一个精确解码的操作批次**；
2. 发布批次；
3. 本地与远程消费者各自应用批次，得到相同的值；
4. 断线/替换后 replica 变「不 ready」，直到重新水合。

**为什么用操作批次而不是「发送整个新状态」**：状态可能很大（会话列表、文档），全量同步成本高。**为什么不用可变共享**：跨进程无法共享内存，而且可变共享无法检测非法写入。

这套设计的代价写在 `delta/README.md` 里，非常诚实：

> Immutability is an ownership contract. **Nothing is frozen or defensively copied**, so an illegal mutation is not detected. It silently corrupts state.

也就是说：为了性能，pi 不用 `Object.freeze`，也不做防御性拷贝。**正确性靠约定与文档**，而不是运行时检查。这是理解 chord 的关键取舍。

## 6. Delta：操作批次的具体实现

`packages/chord/src/delta/`：

```
tracker.ts                  # track(root) → 追踪变更
draft.ts                    # draft 代理（只在 change 期间有效）
diff.ts                     # 计算最小/结构化差异
apply-immutable-trusted.ts  # 应用已信任的批次（快路径）
revision-validator.ts       # 验证不可信批次（远程来的）
README.md                   # 所有权契约表
```

`delta/README.md` 有一张**变更权表**，明确每个值「谁能改」：

| 值 | 可修改？ |
|---|---|
| 传给 `track()` 的根 | 否（所有权转移给 tracker） |
| `change.state` 及其句柄 | 仅在 change 打开期间 |
| 赋值给 draft 的外部对象 | 可以（draft 存的是校验过的克隆） |
| `tracker.value` / `prepared.base` | 否 |
| `prepared.value` / `prepared.ops` | 否 |
| replicated state 的值 | 否 |

这种「用文档表格表达所有权」的做法，是在**没有 Rust 借用检查器的 TypeScript 里模拟所有权语义**。它在性能上赢了，在安全上需要纪律。

批次保证（README）：

> Batches guarantee convergence but are **not canonical or necessarily minimal**.

即：不同路径产生的批次可能不同，但应用到同一 base 上会得到同一结果（收敛）。

## 7. 远端服务：不规定传输

> Bindings carry logical calls and subscriptions through an **application-supplied adapter**. Chord requires strict-JSON arguments, results, snapshots, updates, and catalogues, but **does not prescribe framing, routing, transport, or an application wire envelope**.

这解释了 `protocol` / `client` / `server` 为什么是独立的包（第 24 章）：**Chord 只定义语义，传输由应用选**。

配套类型：
- `JsonRepresentation<T>`：为「含有未知 payload 的应用数据」推导出 wire-safe 类型；
- `isJsonValue()` / `copyJson()`：在适配器边界校验收到的值。

## 8. Context：取消的传播

`@earendil-works/chord/context` 提供类 Go 的 context：

- 取消信号沿调用树传播；
- 可以在 context 上携带「调用作用域内的值」（比如权限、telemetry）而不让 chord 依赖它们。

`BACKGROUND_CONTEXT` 是根 context（`pi-durable` 的 README 就用它）。**为什么需要**：跨进程调用必须能取消（用户按 Esc → 通知远端停止）。JS 的 `AbortSignal` 只解决单进程，chord 的 context 解决「跨 adapter 的取消」。

## 9. 本章要点

- Chord 是**独立的**应用组合运行时，不是 pi CLI 的一部分。
- Facet = 插件的可部署切片；host 校验依赖图、按依赖顺序激活/释放。
- Service 是稳定门面，provider 可断可换；复制状态用「不可变值 + 操作批次」保证收敛。
- Delta 用操作批次而非全量同步；**不做冻结/防御拷贝**，正确性靠所有权契约。
- Chord 只定义语义（strict JSON），不规定传输——所以有了 protocol/client/server 三个包。

下一章：[24 protocol / client / server](24-protocol-client-server.md)
