# 18 资源、设置与包管理

pi 的可配置面很大。这一章把它整理成三层：**目录约定 → 设置合并 → 包分发**。

## 1. 两个配置根

```
<agent-dir>              默认 ~/.pi/agent（PI_CODING_AGENT_DIR 可改）
<project>/.pi            在工作目录下（需项目信任，唯一例外是 sessionDir）
```

| 路径 | 职责 |
|---|---|
| `<agent-dir>/settings.json` | 用户设置：偏好、默认值、资源路径、Pi packages 声明 |
| `<agent-dir>/keybindings.json` | 键绑定 |
| `<agent-dir>/mcp.json` | 全局 MCP 服务器 |
| `<agent-dir>/models.json` | 兼容端点、模型覆盖、自定义 provider |
| `<agent-dir>/auth.json` | 凭证（API key / OAuth） |
| `<agent-dir>/AGENTS.md`, `SYSTEM.md`, `APPEND_SYSTEM.md` | 用户级上下文与提示词 |
| `<agent-dir>/extensions/`, `skills/`, `prompts/`, `themes/` | 用户级资源 |
| `<agent-dir>/sessions/` | 会话文件（第 13 章） |

项目级同名结构在 `.pi/` 下，**且项目级 SYSTEM.md / APPEND_SYSTEM.md 覆盖用户级**（第 15 章）。

**读这段代码时的一个坑**：`config.ts` 里所有资源路径都必须通过 helper 取（`getAgentDir()`、`getPackageDir()`、`getDocsPath()`），因为「源码运行 / npm 安装 / 单文件二进制」三种情况的相对路径不同。`AGENTS.md` 有硬性规定：

> In `packages/coding-agent`, resolve package assets through helpers in `src/config.ts`. Do not use `__dirname` directly.

## 2. 设置合并：三层 + scope

`SettingsManager`（1533 行）合并：

```
默认值  →  全局 settings.json  →  项目 .pi/settings.json  →  CLI 参数
```

它处理的细节比想象中多（这也是它 1533 行的原因）：

- **模型 scope**（哪些模型在某目录下可用）；
- **provider 去重与优先级**；
- **环境变量插值 / 配置值解析**（`resolve-config-value.ts`）；
- **诊断信息**（`collectSettingsDiagnostics` + `deduplicateDiagnostics`，用于「你的配置里有个字段已经废弃」这类提示）；
- **迁移**（`migrations.ts` + `showDeprecationWarnings`）；
- **热重载**（`reload()`，`/reload` 触发）。

## 3. 资源发现：`ResourceLoader`

`core/resource-loader.ts`（1275 行）负责「找出这次运行有哪些资源可用」：

```
discover()
 ├─ extensions      （文件/目录/包 + 内置扩展）
 ├─ skills          （frontmatter 解析 + ignore 规则）
 ├─ prompt templates（*.md → 斜杠命令）
 ├─ themes          （JSON 主题 + 校验器）
 ├─ context files   （AGENTS.override.md / AGENTS.md / CLAUDE.md …）
 └─ MCP servers     （mcp.json，项目 + 全局合并）
```

产出的 `ResolvedPaths` / `ResolvedResource` / `ResourceCollision` / `ResourceDiagnostic` 很有信息量：

- **`ResourceCollision`**：两个来源定义了同名资源（比如用户和项目都有 `/pr` 命令），需要决定覆盖规则；
- **`ResourceDiagnostic`**：某个文件解析失败，但不至于中断启动。

**这是典型的「宽容加载」策略**：一个坏掉的 skill 不该让 `pi` 起不来，但必须让用户知道。

## 4. Pi packages：把资源打包分发

`core/package-manager.ts`（2760 行）+ `package-manager-cli.ts`（1119 行）实现了一套**迷你包管理器**。

### 四种来源

| 来源 | 示例 | 行为 |
|---|---|---|
| npm | `npm:@example/pi-tools@1.0.0` | 装到 pi 的 npm 目录 |
| git | `git:github.com/example/pi-tools@v1` | clone 并对齐到指定 ref |
| URL | `https://github.com/example/pi-tools` | 当作 git 源 |
| 本地 | `./pi-tools` | 直接加载，不复制 |

```bash
pi install npm:@example/pi-tools@1.0.0
pi install git:github.com/example/pi-tools@v1
pi install ./local-package
pi list
pi remove <source>
pi update --extensions
```

### 一个包长什么样

```
my-pi-package/
├── package.json      ← 可选 "pi" 字段声明显式路径
├── extensions/
├── skills/
├── prompts/
└── themes/
```

即「约定目录」优先，`package.json` 的 `pi` 键可以覆盖。

### 安装位置与信任

```
个人安装 → 写入 ~/.pi/agent/settings.json
项目安装 → 写入 .pi/settings.json（--local / -l）
```

文档里有一条关键警告：

> Project packages are installed and loaded only after project trust is resolved. Packages can execute extension code and can include skills that instruct the model to run programs.

所以顺序永远是：**解析信任 → 读项目包声明 → 安装/加载 → 执行**。

### 为什么它值得 2760 行

因为一个「包管理器」需要处理：semver（`gt`/`maxSatisfying`/`satisfies`）、git ref 解析（`parseGitUrl`）、npm registry 交互、缓存目录、文件复制与校验（`createHash`）、`ignore` 规则、并发安装、进度回调、损坏恢复、`package-lock` 风格的版本对齐。

`package.json` 里能看到依赖：

```json
"ignore": "...", "minimatch": "...", "semver": "..."
```

**这也是「pi 不只是个 CLI，而是一个平台」的证据**：它有资源格式、分发机制、版本管理、以及 `pi install` 这样的子命令。

## 5. 与「供应链加固」的关系

根 README 有一整节 "Supply-chain hardening"，在本仓库的语境里非常相关：

- 直接外部依赖**钉死精确版本**（`save-exact=true`），内部 workspace 包用 range；
- `.npmrc` 设 `min-release-age=2`，避免当天发布的依赖被立即解析；
- `package-lock.json` 是唯一真相，pre-commit 阻止意外提交（除非 `PI_ALLOW_LOCKFILE_CHANGE=1`）；
- `npm run check` 校验 pinned deps、TS import 兼容性、生成的 install-lock；
- pi.dev 安装器用 `packages/coding-agent/install-lock/` 固定传递依赖（npm 包本身不固定）；
- 生命周期脚本需要显式 allowlist（`generate-coding-agent-install-lock.mjs` 里）。

对一个会执行任意代码的 agent 工具，这种程度的谨慎是必要的。

## 6. 三种「安装形态」与自更新

`config.ts` 里的 `detectInstallMethod()` 返回：

```ts
type InstallMethod = "bun-binary" | "npm" | "pnpm" | "yarn" | "bun" | "unknown";
```

据此生成自更新命令（`SelfUpdateCommand`，可多步：先卸载再安装）。`utils/windows-self-update.ts` 还有 Windows 特有的隔离区清理。

**这意味着 `pi update` 的实际行为取决于你是怎么装的**——这是一个容易被忽略但很实际的工程点。

## 7. 依赖与运行时约束

根 `package.json`：

```json
"engines": { "node": ">=22.19.0" },
"type": "module"
```

以及一个有意思的约束（`AGENTS.md`）：

> Use only erasable TypeScript syntax (Node strip-only mode) in code checked by the root config: no parameter properties, `enum`, `namespace`/`module`, `import =`, `export =`.

**为什么**：Node 22 可以「剥离类型直接运行 TS」（`--experimental-strip-types`，后来稳定）。要求「可擦除语法」意味着源码可以**不编译直接跑**（`pi-test.sh` 就这么干的）。这直接提升了开发体验和启动速度。

## 8. 本章要点

- 两个配置根：`~/.pi/agent`（用户）与 `<repo>/.pi`（项目，需信任）；资源路径必须走 `config.ts` 的 helper。
- 设置是四层合并（默认 → 全局 → 项目 → CLI），并带诊断与迁移。
- `ResourceLoader` 宽容加载：坏资源给诊断，不阻塞启动。
- Pi packages 是完整的迷你包管理器（npm/git/本地），因为 pi 是「有资源格式的平台」。
- 供应链加固、安装方式探测、自更新命令、可擦除 TS 语法，都是同一套「多形态分发」的现实约束。

下一章：[19 四种运行模式](19-coding-agent-四种运行模式.md)
