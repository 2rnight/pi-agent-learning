# 24 protocol / client / server：把 agent 变成远程服务

三个包合起来是一条完整的「远程会话」链路：

```
client ──(字节流)──► server ──► session worker（真正跑 agent 的进程）
   ▲                    │
   └── 共用 pi-protocol 的 CBOR 信封与分帧 ──┘
```

它们都标注 experimental，但设计细节很完整。

## 1. pi-protocol：只做信封与分帧

> Runtime-neutral routed envelopes, CBOR encoding, and byte-stream framing.

### 协议版本 8 提供的能力

- 版本握手，识别逻辑 `serverId`；
- 明确的两种目标：**server** 与 **Session**；
- 带关联 id 的请求/响应，payload 是不透明 strict-JSON；
- 请求取消、不透明订阅更新、**带外（out-of-band）附着变更**；
- 非空错误码、有界的传输消息。

路由标识：

```
server target  = { serverId }
Session target = { serverId, sessionId, attachmentId }
```

三者组合把调用「围栏」到一个逻辑服务器 + 一个持久 Session + 一个实时展示附着。

### 为什么要 `attachmentId`

一个 Session 可以有**多个展示端**（终端 + WebUI 同时看）。`attach()`/`detach()` 是管理操作，**不返回路由 id**；服务器通过带外 `attachment` 消息告知选中的实时路由。这样：

- 业务结果里不会泄漏路由标识；
- 断线只释放**那一个展示**的附着（在已接受的调用结算之后）；
- 同一连接重复 `attach` 是幂等的。

### 分帧格式（最简单也最重要）

```
┌────────────────┬─────────────────────────────┐
│ 4 字节 BE 长度  │  一个 definite-length CBOR  │
└────────────────┴─────────────────────────────┘
```

```ts
const hello: ClientHello = { type: "hello", version: PROTOCOL_VERSION };
transport.send(encodeClientMessage(hello));

const decoder = new ServerMessageDecoder({ maxFrameLength: 1024 * 1024 });
for (const message of decoder.push(incomingChunk)) handleServerMessage(message);
decoder.end();
```

**解码器接受任意分片与合并**（`push(chunk)` 返回 0..N 条消息）。这是字节流协议最基本的要求，也是最容易写错的地方。

### 默认限制（安全阀）

| 限制 | 默认值 |
|---|---|
| 单个 CBOR payload / frame | 16 MiB |
| 数组元素 / map 条目 | 1,000,000 |
| 嵌套层级 | 64 |

并且：**所有 envelope schema 拒绝未知属性**，codec 递归拒绝非 JSON 的不透明 payload（非有限数、字节数组、`undefined`、原型、循环引用）。违规抛 `ProtocolValidationError`。

**严格拒绝未知字段**是一个「向前不兼容但更安全」的选择：解析器不会被未预期字段带偏。

### 一个清晰的边界声明

> Chord owns the payload semantics... `pi-protocol` validates that each opaque payload is strict JSON but **does not validate or export its Chord grammar**.

即：协议层只保证「这段 JSON 是合法 JSON」，不懂 `{ serviceId, instance, member, args }` 的含义。**分层清晰，可替换。**

同样是明确不管的：**peer 认证**。实验性传输不实现身份认证（见第 26 章的安全章节）。

## 2. pi-client：传输无关

```ts
const transportFactory: ByteTransportFactory = async (handlers) => {
  return {
    async send(chunk) { /* 按调用顺序投递字节，遵守背压 */ },
    close() {},
  };
};

const client = await Client.connect({ serverId, transportFactory });
const result = await client.request(
  { serverId: client.hello.serverId },
  { serviceId: "example.service", member: "read", args: [] },
);
```

要点：

- **只要「有序字节传输」**：WebSocket、Unix socket、任何 stream 都行；
- 客户端校验物理端点的 `serverId` 与期望一致（防止连错服务器/陈旧 socket）；
- `request()` 返回 Promise，内部做 id 关联；
- 取消 = 发 cancel 帧（不是断连接）。

`client/src/unix.ts` 提供 Unix socket 的具体实现（`pi-client/unix`）。

## 3. pi-server：不透明的路由器

> Experimental local server that routes clients to application-hosted durable Sessions.

**它最激进的设计是「不透明」**：

> Its `invokeService()` forwards an opaque service/member envelope to the selected Session endpoint; **the server validates the attachment route but does not load the facet contract**.

也就是说服务器**不理解**业务 payload。它只做三件事：

1. 校验路由（serverId/sessionId/attachmentId 是否存在且匹配）；
2. 转发；
3. 管理附着的生命周期。

对应的一串设计点（README 原文）：

- server service 调用与订阅通过连接的 `RoutedServerServiceAttachment` 不透明路由；
- 应用自己拥有的 `SessionDirectory` 把私有目录投影成「可展示的安全状态」；
- 应用自己拥有的 `SessionManagement` 创建/删除/附着/分离 Session，**业务结果里不暴露路由 id**；
- 附着变更在路由安装/清除之后带外发布；
- Session 调用通过 `invokeService` 路由，**服务器端不解码业务 payload**；
- 订阅更新保持在请求它的附着范围内；
- 应用观察（如 transcript）作为普通服务状态路由，没有服务器自有的业务 schema。

**为什么这么设计**：服务器是可替换的（`experimental/` 里有 coordinator + worker 的模型）。如果服务器理解业务，那么「升级服务器的业务逻辑」就会和「运行中的 worker」冲突。保持不透明，服务器就只是管道。

### Session 与 worker

- 「durable Session 和它的 Harness 保持在 Session worker 进程内本地」；
- 一个 Session 可以有多个展示附着；
- 连接丢失：**本地响应被 reject，但附着要等到已接受的调用结算才释放**；
- host 决定「零展示需求 + worker 本地无活动」时回收 worker；
- server 关闭时关闭所有 routed Session handle。

这解释了为什么 `durable` 要存在（第 25 章）：**worker 必须能在任何时候被杀死并恢复**，所以 Session 的状态必须先落盘。

## 4. 三者的分工（一张表）

| | protocol | client | server |
|---|---|---|---|
| 知道 CBOR 格式 | ✅ | ✅（通过 protocol） | ✅ |
| 知道分帧 | ✅ | ✅ | ✅ |
| 知道 Chord service 语义 | ❌ | ⚠️（应用层） | ❌ |
| 管附着路由 | 定义字段 | 使用 | ✅ 校验 |
| 管 Session 生命周期 | ❌ | ❌ | ✅（通过应用回调） |
| 依赖 chord | ✅（类型上） | ✅ | ✅ |

依赖图印证：`protocol → chord`、`client → {chord, protocol}`、`server → {chord, protocol}`。

## 5. 与 pi CLI 的关系

pi 的 RPC 模式（第 19 章）用的是**另一套协议**：stdin/stdout 上的 JSONL（`RpcCommand` / `RpcResponse`）。

两套并存说明：

| | RPC 模式 | protocol/client/server |
|---|---|---|
| 传输 | 进程 stdin/stdout | 任意有序字节流 |
| 编码 | JSONL | CBOR + 长度前缀 |
| 目标 | 控制**一个** pi 进程 | 多会话、多展示、可附着/分离 |
| 成熟度 | 产品功能 | experimental |

也就是说：**单机控制用 RPC，分布式/多客户端用 chord 那套**。

## 6. 本章要点

- 协议 = 4 字节 BE 长度 + definite-length CBOR；解码器必须处理任意分片。
- 两种目标（server / Session），Session 目标带 `attachmentId` 支持多展示端；附着变更走带外消息。
- 严格校验：拒绝未知字段、拒绝非 JSON payload，16 MiB / 100 万元素 / 64 层上限。
- client 只要「有序字节传输」；server 是**不透明路由器**，不理解业务 payload——为了可替换性。
- 协议层不做 peer 认证（实验阶段）；与 CLI 的 RPC（JSONL）是两套不同用途的协议。

下一章：[25 durable 与 env](25-pi-durable与pi-env.md)
