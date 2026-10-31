# 在各平台接入做题 sub-agent（Solver）

本 skill 的「盲做 + 物理隔离」依赖一个独立 sub-agent 跑 `agents/solver.md`。本指南说明如何把这份定义接到不同运行时的 agents 体系里。**文件正文（角色 / 合约 / 信息隔离 / 硬约束）平台无关，可直接复用；需要按平台调整的是文件放置位置与 frontmatter 格式。** 各平台字段名会随版本变化，落地时以官方文档为准。

## 为什么必须独立 sub-agent

同一 LLM 里「扮演学生」极易退化成「出题者猜学生」。只有把 solver 放进**全新上下文**，让它物理上读不到出题 agent 的题解与预期，行为证据才可信（详见 README「运行环境要求」）。提示词盲做是降级方案，独立性打折。

## 通用接入三步

1. **放文件**：把 `agents/solver.md` 复制到目标平台的 agents 目录（见下表），文件名通常即 agent 名（`solver`）。
2. **对齐 frontmatter**：按平台 schema 调整头部字段（见各平台示例）。
3. **隔离式调用**：出题 agent 调用 solver 时，**只传【题目】原文 +【角色设定】**，严禁带题解、标准答案、预期坑点。这一步由调用方保证，平台无关。

## 各平台接入对照

| 平台 | Agent 文件位置 | frontmatter 格式 | 工具权限控制 | 备注 |
|---|---|---|---|---|
| **Claude Code** | `.claude/agents/`（项目）/ `~/.claude/agents/`（用户） | YAML：`name` / `description` / `tools` / `model` | 显式 `tools` 列表 | `solver.md` 默认即为该格式，直接可用 |
| **CodeBuddy** | 任意路径，如 `.codebuddy/agents/solver.md` | YAML：`name` / `description` / `tools` | 显式 `tools` 列表 | 通过 Task 工具 `subagent_path` 指向该文件 |
| **Cursor** | `.cursor/agents/`（项目）/ `~/.cursor/agents/`（用户）；兼容 `.claude/agents/`、` .codex/agents/` | YAML：`name` / `description` / `model` / `readonly` / `is_background` | **无 `tools` 字段，自动继承父代理全部工具** | 同名时 `.cursor/` 优先于 `.claude/`/`.codex/` |
| **OpenCode** | `.opencode/agents/`（项目）/ `~/.config/opencode/agents/`（全局） | YAML：`description` / `mode: subagent` / `model` / `temperature` / `tools` / `maxSteps` | 显式 `tools: {read, write, edit, bash}` | 文件名即 agent 名 |
| **Gemini CLI** | `.gemini/agents/*.md`（项目） | YAML：`name` / `description` / `model` / `max_turns` / `timeout_mins` / `tools` | 显式 `tools` 列表 | 各 sub-agent 独立上下文；默认已启用 |
| **GitHub Copilot** | `.github/agents/<name>.agent.md`（仓库）/ 用户配置目录 | YAML：`name` / `description` / `tools` / `model` / 代理连接 | 显式 `tools` 列表 | VS Code 自动发现 `.github/agents/` 下 `.agent.md` |
| **Windsurf (Codeium)** | `.agents/agents/`（项目根，Agent Skills 规范）；家目录 `~/.codeium/windsurf` | 同 Agent Skills 规范（YAML frontmatter） | 按规范显式声明 | 自动发现，亦兼容 `.windsurf/` 旧目录 |
| **OpenAI Codex** | `.codex/agents/`（项目）/ `~/.codex/agents/`（用户） | **非 markdown**：TOML / `agents/openai.yaml`（Agent Skills 规范） | 在 TOML/YAML 内声明 | **不直接读 markdown agent**，需把正文转为 Codex 的 agent spec（见下） |
| **DeepSeek Harness (DSH)** | `~/.dsh/agents/`（全局）/ 项目 `.dsh/agents/`（或角色插件用 `.dsh/roles/`、`~/.dsh/roles/`） | YAML：`name`(必填) / `description` / `display_name` / `model` / `thinking` / `deep` / `background` / `maxRounds` | 基础 frontmatter **无 `tools` 字段（工具继承）**；角色插件（`dsh-plugin-subagent-roles`）另提供 per-role 工具过滤与预算 | 需装 `dsh-subagent-registry` 或 `dsh-plugin-subagent-roles` 插件；正文原样作 system prompt |

> ⚠️ 上表为各平台公开文档的当前形态，字段名与目录随时可能调整，**以对应平台官方文档为准**。

## 各平台 frontmatter 示例

**Claude Code / CodeBuddy**（即 `agents/solver.md` 原生格式）：

```yaml
---
name: solver
description: 做题 agent。扮演目标人群的典型学生，在隔离上下文中盲做题目，产出第一人称行为证据。必须以独立 subagent 调用。
tools: Read, Write, Bash, Grep, Glob
---
# 以下正文与 agents/solver.md 一致
```

**Cursor**（去掉 `tools`，工具自动继承；如只需只读可加 `readonly: true`，但 solver 要写报告故保持默认）：

```yaml
---
name: solver
description: 做题 agent。扮演目标人群的典型学生，在隔离上下文中盲做题目，产出第一人称行为证据。
model: inherit
---
# 以下正文与 agents/solver.md 一致
```

**OpenCode**（`.opencode/agents/solver.md`）：

```yaml
---
description: 做题 agent。扮演目标人群的典型学生，在隔离上下文中盲做题目，产出第一人称行为证据。
mode: subagent
model: inherit
tools:
  read: true
  write: true
  edit: true
  bash: true
---
# 以下正文与 agents/solver.md 一致
```

**Gemini CLI**（`.gemini/agents/solver.md`）：

```yaml
---
name: solver
description: 做题 agent。扮演目标人群的典型学生，在隔离上下文中盲做题目，产出第一人称行为证据。
model: inherit
max_turns: 12
tools:
  - read
  - write
  - bash
---
# 以下正文与 agents/solver.md 一致
```

**GitHub Copilot**（`.github/agents/solver.agent.md`）：

```yaml
---
name: solver
description: 做题 agent。扮演目标人群的典型学生，在隔离上下文中盲做题目，产出第一人称行为证据。
tools:
  - read
  - write
  - bash
---
# 以下正文与 agents/solver.md 一致
```

**OpenAI Codex**（需转换格式，不直接用 markdown）：Codex 不消费 markdown agent 文件，而是读取 TOML 或 Agent Skills 规范的 `agents/openai.yaml`。把 `agents/solver.md` 的**正文**放进 Codex agent spec 的 prompt / 指令字段，并在其中声明 `name`、`description` 与工具权限。具体字段写法以 [Codex 官方文档](https://developers.openai.com/codex/) 为准。

**DeepSeek Harness（DSH）**：先装 `dsh-subagent-registry`（或 `dsh-plugin-subagent-roles`）插件，再把文件丢进 `~/.dsh/agents/solver.md`（全局）或项目 `.dsh/agents/solver.md`。注意去掉 Claude Code 风格的 `tools` 字段、改用 DSH 的字段；`deep: 0` 让 solver 成为叶子（不再派生子代理，利于隔离），`maxRounds` 约等于其他平台的 `max_turns`：

```yaml
---
name: solver
description: 做题 agent。扮演目标人群的典型学生，在隔离上下文中盲做题目，产出第一人称行为证据。
model: inherit
deep: 0
maxRounds: 12
---
# 以下正文与 agents/solver.md 一致（原样作为 system prompt）
```

> 工具权限：基础 registry 插件下 solver 继承运行时的全部工具；若要收窄，改用 `dsh-plugin-subagent-roles` 并配置 per-role 工具过滤与预算。关键不变式仍是只给「读题 + 算题 + 写报告」的工具。

## frontmatter 字段差异要点

- **`tools` 字段**：Claude Code / CodeBuddy / OpenCode / Gemini CLI / GitHub Copilot 都支持显式 `tools`；**Cursor 不支持**（sub-agent 自动继承父代理的全部工具，无法在 frontmatter 收窄）；Codex 不用 markdown，权限在其 TOML/YAML 内声明。
- **关键不变式**：无论哪个平台，solver 都应**只给读题 + 算题 + 写报告**的工具，**不给**网络访问或出题文件的读取权限，否则隔离可能被绕过。
- **`model`**：默认 `inherit`（沿用父代理模型）即可；如需更便宜/更强的模型可显式指定。

## 调用约定（最关键）

无论平台，出题 agent 调用 solver 时必须遵守：

- 输入**只有**两项：`【题目】原文` + `【角色设定】`。
- **绝不**传入：题解、标准答案、自检报告、深挖记录、预期陷阱位置、「这题想考什么」。
- solver 只产出第一人称行为报告，**判定权在出题 agent 的对账环节**。

## 验证隔离是否生效

跑一次后检查 solver 输出：若它提到「我看到答案了」「这题的题眼是…」「这题想考…」，说明隔离被破坏（调用方把出题意图带进去了，或它读到了出题文件）。健康输出应只有「我先写了…」「我卡在…」「题面矛盾在…」。
