# 06 认证与凭证

pi 支持两类凭证，覆盖「付费用 API」和「用订阅额度」两种现实：

1. **API Key**（含环境变量、`auth.json`、CLI 覆盖）；
2. **OAuth**（ChatGPT/Codex 订阅、GitHub Copilot、Anthropic、xAI、OpenRouter、Kimi、Radius…）。

这一章讲它们的存储、解析顺序、刷新时机。

## 1. 目录结构

```
packages/ai/src/auth/
├── types.ts              # Credential / ProviderAuth / AuthResult / AuthContext
├── credential-store.ts   # 存储抽象 + InMemoryCredentialStore
├── resolve.ts            # 解析顺序与刷新逻辑（核心）
├── context.ts            # 认证上下文（谁在问、什么环境）
├── helpers.ts
└── oauth/
    ├── pkce.ts           # PKCE 工具
    ├── callback-server.ts # 本地回调服务器（浏览器授权）
    ├── device-code.ts    # 设备码流程（无法打开浏览器时）
    ├── meta.ts           # OAuth provider 元数据
    ├── load.ts           # 动态加载
    ├── anthropic.ts / openai-codex.ts / openai-chatgpt.ts
    ├── github-copilot.ts / xai.ts / openrouter.ts / kimi-coding.ts / radius.ts
    └── ...
```

**OAuth 是按供应商写的，不是通用框架**。这不是偷懒——每家（尤其 Copilot、Codex）的 token 交换、刷新、header 注入都高度定制。

## 2. 存储：Provider 独占到 per-provider 串行化

`credential-store.ts`：

```ts
export interface CredentialStore {
  read(providerId): Promise<Credential | undefined>;
  list(): Promise<readonly CredentialInfo[]>;
  modify(providerId, fn: (current) => Promise<Credential | undefined>): Promise<Credential | undefined>;
  delete(providerId): Promise<void>;
}
```

两个设计决策：

1. **一个 provider 一份凭证**（keyed by `Provider.id`）。不做「多账号切换」——那是扩展/上层的事。
2. **写操作按 provider 串行**：`modify()` 内部用 promise 链把同一 provider 的修改排队：

```ts
private async enqueue<T>(providerId, task, options): Promise<T> {
  const previous = this.chains.get(providerId) ?? Promise.resolve();
  const queued = (async () => { await previous.catch(() => {}); signal.throwIfAborted(); return task(); })();
  // ...
}
```

**为什么要串行**：OAuth 刷新是「读 → 交换 → 写」的复合操作。两个并发请求同时发现 token 过期，会各自去刷新，后写的覆盖先写的，甚至把 refresh token 用废。串行化让刷新天然幂等。

`Coding-agent` 用的是持久化实现 `AuthStorage`（`core/auth-storage.ts`），落在 `<agent-dir>/auth.json`；`ReadOnlyAuthStorage` 用于不该改配置的场景。

## 3. 解析顺序：一条明确的优先级

`resolve.ts` 的顶部注释就是规则：

> A stored credential owns the provider: ambient/env is consulted only when nothing is stored. No silent env fallback after a failed refresh or for a credential type without a matching handler.

翻译成顺序：

```
1. 显式 override（本次调用传入的 apiKey / env）
2. CredentialStore 里的凭证
   2a. API key → 直接用
   2b. OAuth    → 检查有效期，必要时刷新
3. 环境变量（provider 声明的变量名，见 env-api-keys.ts）
4. 都没有 → 返回 undefined，上层给出 /login 引导
```

两个「反直觉」的约束，都是为了避免静默错误：

- **存了凭证就不再回落到环境变量**：如果你 `--login` 了一个账号，但 token 刷新失败，pi 不会「偷偷」用你 shell 里的另一个 key 去发请求（那会导致账单/数据发错地方）。
- **凭证类型必须匹配 handler**：存的是 OAuth 但 provider 只实现了 api-key handler，就是错误，不做兜底。

### 提前刷新：`minOAuthValidityMs`

```ts
export interface AuthResolutionOverrides {
  apiKey?: string;
  env?: ProviderEnv;
  /** Require this much remaining OAuth-token validity; defaults to five minutes. */
  minOAuthValidityMs?: number;
  signal?: AbortSignal;
}
```

默认「少于 5 分钟就刷新」。**这是长会话的必备设计**：一次 agent run 可能持续几十分钟、几十次请求，不能等到 401 才发现过期。

## 4. 为什么每轮循环都重新取 key

回看第 02 章那段代码：

```ts
// Resolve API key (important for expiring tokens)
const resolvedApiKey =
  (config.getApiKey ? await config.getApiKey(config.model.provider) : undefined) || config.apiKey;
```

`agent-loop.ts` 里的注释写得很直白：*important for expiring tokens*。所以 **API key 是每次请求前解析的，不是构造 Agent 时固定的**。

这条设计把「认证」和「会话」解耦：会话可以跨 token 生命周期存活，甚至可以在运行中切换账号（扩展改 store 即可）。

## 5. OAuth 的两种交互形态

| 形态 | 文件 | 适用 |
|---|---|---|
| 本地回调服务器 | `callback-server.ts` | 有浏览器（桌面终端） |
| 设备码 | `device-code.ts` | 无浏览器（SSH、容器、手机） |

`pkce.ts` 提供 PKCE（code verifier/challenge），避免公共客户端用 client secret。`load.ts` 负责「按 provider id 找到对应的 oauth 实现」——这里也是**懒加载**：没登录过的 provider 不需要解析 OAuth 代码。

## 6. 从 CLI 视角看认证

`coding-agent/src/cli/` 里有专门的文件：

- `auth-command.ts`：解析 `pi auth ...` 子命令（帮助、参数校验、错误类型）
- `auth-check.ts`：`checkProviderAuth()`、`getProviderCredential()`
- `credential-print.ts`：打印当前凭证来源（不含密钥本体）
- `startup-ui.ts`：首启动的登录引导

`main.ts` 里有一段值得注意的实现：认证子命令在**解析完整 CLI 之前**就被截获（`runAuthCommand`），因为登录流程可能需要在没有模型、没有项目的情况下运行。

## 7. 安全边界（值得单独强调）

- `auth.json` 是明文凭证文件，权限由操作系统保证；
- pi **不提供凭证的进程内隔离**：扩展与工具以同权限运行（`SECURITY.md` / `README.md` 都明确说了「Pi does not include a built-in permission system」）；
- 想隔离就用容器/micro-VM，见 `docs/containerization.md`（Gondolin / Docker / OpenShell 三种模式）。

对读者的实际建议：**在不可信仓库里跑 pi 之前，先读 `packages/coding-agent/docs/security.md`。**

## 8. 本章要点

- 凭证两种：API key 与 OAuth；统一进 `CredentialStore`，按 provider 一份。
- **写操作 per-provider 串行**，让 OAuth 刷新天然不冲突。
- 解析顺序：显式 override → 存储凭证 → 环境变量；**存储凭证存在时不回落**。
- 每次请求前重新解析 key（默认提前 5 分钟刷新），让长会话不受 token 过期影响。

下一章：[07 Agent 与状态](07-agent-core-Agent与状态.md)
