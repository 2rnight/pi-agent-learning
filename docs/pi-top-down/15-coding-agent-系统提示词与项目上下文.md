# 15 系统提示词与项目上下文

pi 的系统提示词不是一段硬编码字符串，而是**由若干具名分节拼出来的**，并且可以被文件、设置、扩展三层覆盖。

## 1. `buildSystemPrompt()` 的输入

`core/system-prompt.ts` 的 `BuildSystemPromptOptions`：

```ts
export interface BuildSystemPromptOptions {
  customPrompt?: string;            // 替换默认前缀
  forceSystemPrompt?: string;       // before_agent_start 里的强制全量替换
  selectedTools?: string[];         // 默认 [read, bash, edit, write]
  toolSnippets?: Record<string, string>;   // 每个工具的一句话说明
  toolGuidelines?: Record<string, string[]>; // 每个工具贡献的「规则条目」
  promptGuidelines?: string[];      // 追加规则
  appendSystemPrompt?: string;      // 用户 APPEND_SYSTEM.md
  sections?: Record<string, string>;// 额外 XML 包裹的分节
  cwd: string;
  contextFiles?: Array<{ path: string; content: string }>;
  skills?: Skill[];
}
```

注意三个「工具贡献」字段：`toolSnippets`、`toolGuidelines`、`promptGuidelines`。

**这是 pi 一个很聪明的做法**：每个工具文件自己导出它希望出现在提示词里的文案。比如 `read.ts`：

```ts
export const readToolSystemPromptContribution = {
  snippet: "Read file contents",
  guidelines: ["Use read to examine files instead of cat or sed."],
} as const;
```

好处：**工具新增/删除时提示词自动同步**，不会出现「提示词里让你用 grep，但这个构建里没有 grep」的错位。扩展注册工具时也能贡献自己的说明（第 17 章）。

## 2. 分节结构：为什么用 XML 标签

```ts
const SYSTEM_PROMPT_SECTION_NAME = /^[a-z][a-z0-9_-]*$/;

// buildSystemPromptSections：
const sections: SystemPromptSections = { preamble: promptSections.preamble };
for (...) {
  if (name !== "preamble") sections[name] = `<${name}>\n${content}\n</${name}>`;
}
```

输出形态：

```text
（preamble：不带标签的正文）
<project_instructions path="/repo/AGENTS.md">
...
</project_instructions>
<skills>
...
</skills>
<cwd>/repo</cwd>
```

两个原因：

1. **模型能定位**：后来追加的 system 消息说「`<skills>` 这一段换成新的」，模型能精确对应上（第 10 章的 `sections` 补丁就是这个机制）。
2. **可解析**：扩展可以按标签做 markdown 变换（`MarkdownTransformer`）。

约束：分节名必须是 `^[a-z][a-z0-9_-]*$` 且不能叫 `preamble`（`system-prompt.ts:137` 有校验）。**不能是数字开头**，因为 JSON 对象会重排整数键。

## 3. 项目上下文：AGENTS.md 的发现顺序

`core/resource-loader.ts:185`：

```ts
const candidates = ["AGENTS.override.md", "AGENTS.md", "AGENTS.MD", "CLAUDE.md", "CLAUDE.MD"];
```

发现规则（结合 `docs/configuration.md`）：

```
1. 用户级：<agent-dir>/ 下的 AGENTS.override.md / AGENTS.md / CLAUDE.md …
2. 项目级：从 cwd 向上查找 AGENTS.md（受 .gitignore/.ignore 影响）
3. 渲染成 <project_instructions path="...">…</project_instructions>
```

支持 `CLAUDE.md` 是刻意的互操作：很多仓库已经有 Claude 的上下文文件，pi 直接复用，不强迫你改名。

**这个仓库本身就是例子**：根目录的 `AGENTS.md`（8.5KB）既是贡献者文档，也是 pi 运行时的上下文输入。文件里那些规则（「不要跑 `git add -A`」「changelog 只写在 Unreleased 下」）同时约束人和 agent——**这是「自我托管」的最直接形态**。

## 4. 三层覆盖：SYSTEM.md / APPEND_SYSTEM.md / customPrompt

优先级（`docs/configuration.md` 明确写了）：

```
项目 .pi/SYSTEM.md   >  用户 <agent-dir>/SYSTEM.md      （替换，不合并）
项目 .pi/APPEND_SYSTEM.md > 用户 <agent-dir>/APPEND_SYSTEM.md（追加，不合并）
```

> For `SYSTEM.md` and `APPEND_SYSTEM.md`, the trusted project file takes precedence over the corresponding agent-directory file. Files with the same name are not combined.

两个注意点：

- **项目级优先**（危险但有用于「这个仓库的 agent 必须这么干」）；
- **同名文件不合并**，避免两段风格冲突的指令打架。

`customPrompt` 则是程序化入口（SDK 里用），`forceSystemPrompt` 是扩展在 `before_agent_start` 里**完全接管**提示词（优先级最高）。

## 5. Skills：按需加载的指令包

`core/skills.ts`：

```ts
const MAX_NAME_LENGTH = 64;
const MAX_DESCRIPTION_LENGTH = 1024;
const IGNORE_FILE_NAMES = [".gitignore", ".ignore", ".fdignore"];
```

Skill 是「一个目录 + 一个带 frontmatter 的入口文件」：

```markdown
---
name: release
description: Prepare and publish a release
---
（完整指令）
```

关键设计：**启动时只把 `name + description` 放进系统提示词**（`formatSkillsForPrompt`），完整内容在模型「调用」该技能时才读入。

为什么这么做：技能可以很多、很长（仓库里有 `add-llm-provider.md`、`release.md`、`interactive-testing.md`）。全塞进系统提示词会浪费几万 token 且稀释注意力。**按需加载是「上下文经济学」的典型应用。**

`skills.ts` 里还有一个细节：它自己解析 `.gitignore/.ignore/.fdignore`（`ignore` 库 + `prefixIgnorePattern`），保证 `skills/` 下被忽略的文件不会被当成技能——**和 `fd`/`rg` 的行为保持一致**。

## 6. Prompt templates：斜杠命令的模板

`core/prompt-templates.ts` + `expandPromptTemplate()`：

```
~/.pi/agent/prompts/*.md       → 用户级模板
<repo>/.pi/prompts/*.md        → 项目级模板
```

仓库自己的 `.pi/prompts/` 就是例子：

```
cl.md  deslop.md  is.md  pr.md  sa.md  wr.md
```

用法：输入 `/pr` → 展开成模板文本 → 变成一个 user message。**模板展开发生在「输入变消息」这一步**（第 02 章第一站），所以模型看到的是展开后的完整指令，而不是 `/pr` 这个 token。

## 7. 提示词里到底写了什么

不必逐字读，但值得知道结构：

```
[角色与总则]
[工具使用规则]        ← 来自 toolGuidelines
[交互风格]
[项目指令]            ← AGENTS.md
[技能列表]            ← 只有 name + description
[环境信息]            ← cwd、平台、shell 等
```

`buildSystemPrompt` 的返回值不是字符串，而是 `{ content: "", sections: {...} }`——**第一条 system 消息的 `content` 为空，全部内容都在 sections 里**。这样后续补丁可以按节替换（`system-prompt.ts:191`）。

## 8. 修改提示词的正确姿势（给使用者的建议）

| 需求 | 做法 | 不要做 |
|---|---|---|
| 加通用规则 | 用户 `AGENTS.md` 或 `APPEND_SYSTEM.md` | 改 `system-prompt.ts` |
| 这个仓库专属规则 | 项目 `AGENTS.md` | 提交到 pi 源码 |
| 完全替换提示词 | `.pi/SYSTEM.md` | fork 仓库 |
| 按需的复杂流程 | 写成 skill | 全塞进 AGENTS.md |
| 可复用的固定指令 | prompt template | 每次手打 |

## 9. 本章要点

- 系统提示词是**具名分节的集合**（XML 包裹），不是一段字符串；这样才支持「按节补丁」。
- 工具自己贡献提示词片段，工具增删时提示词自动同步。
- 项目上下文发现顺序：`AGENTS.override.md` → `AGENTS.md` → `CLAUDE.md`，用户级与项目级都参与。
- `SYSTEM.md`/`APPEND_SYSTEM.md` 项目级覆盖用户级且不合并。
- Skills 只把摘要放进提示词，正文按需加载。

下一章：[16 上下文压缩](16-coding-agent-上下文压缩.md)
