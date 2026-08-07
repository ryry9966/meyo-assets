---
title: 让 Agent 认识你：在 OpenClaw 中落地 USER.md 的用户画像模式
feedId: 32001
source: 综合讨论
publishedAt: 2026-08-07
---

## 背景：Agent 最缺的不是能力，是上下文

基于 OpenClaw、MCP（Model Context Protocol）构建的 Agent 已经能操作文件、调用 API、执行 shell 命令，但一个常见痛点始终存在：**每次对话都要重新交代自己的偏好、项目路径、工具链和约束条件**。

即使设定 system prompt，它仍然是“一次性的”，无法跨对话、跨工具之间共享。当 Agent 需要帮你在多个项目间切换，或与自定义 MCP 服务器协同工作时，这种重复上下文注入会迅速劣化体验。

一种工程化解法是为 Agent 提供一个“用户自述文件”——`USER.md`，类似开源项目里的 `README.md`，但描述的不是项目，而是**你**：你是谁、你的工作环境、习惯、常用配置和自动化规则。

## 核心思路：将个人上下文视为可解析的“配置”

`USER.md` 的本质是 **Agent 可见的静态知识源**。你可以把它放在 `$HOME/.openclaw/USER.md` 或者某个固定路径，通过 MCP 文件系统工具（如 `mcp-server-filesystem`）或自定义插件让 Agent 在需要时主动读取。

我们需要回答三个问题：
1. `USER.md` 写什么？
2. Agent 如何可靠读取？
3. 如何防止腐烂和安全问题？

## 做法：从最小可用版本开始

### 1. 定义文件结构

不要写散文。用结构化 Markdown，方便解析和人类阅读。一个可复制的模板：

```markdown
# User Profile for AI Agents

## Identity
- Name: Alex
- Role: Backend engineer, CLI enthusiast
- OS: macOS / Ubuntu (WSL2)
- Timezone: Asia/Shanghai

## Workspace
- Primary project path: ~/dev/work/platform-service
- Side project path: ~/dev/lab/
- Notes & snippets: ~/Notes/

## Preferences
- Shell: zsh, prefer `ripgrep` over `grep`
- Editor: Neovim, terminal-first workflow
- Git: always sign commits with GPG key `A1B2C3D4`
- Language: Python 3.12 (managed by mise), Go 1.22, TypeScript for quick scripts

## Commands & Aliases
- Build: `task build` (uses Taskfile)
- Lint: `task lint`
- Run local: `task dev`
- Deploy staging: `task deploy-staging` (requires VPN)
- Custom: `prd` opens project root directory

## Constraints
- Never push to `main` directly; always create a feature branch.
- Do not install global npm packages without asking.
- Do not modify dotfiles outside of the workspace unless explicitly requested.
- All generated scripts must set `set -euo pipefail`.
```

### 2. 让 Agent 在启动时“认识你”

在 OpenClaw 的 system prompt 中加入引用指令，而不是把整个文件内容硬编码进去：

```
Before each task, read $HOME/.openclaw/USER.md to understand the user's context. 
If the file does not exist, ask the user to create one. Do not cache the content for more than one conversation without explicit instruction.
```

配合 MCP 文件系统工具，Agent 可以在需要时动态调用 `read_file` 获取 `USER.md`。这样做的好处是热更新——修改文件后无需重启 Agent。

如果你的 OpenClaw 实例支持 `on_session_start` 钩子，也可以在这里触发一次读取，将内容注入到当前 session 的上下文中。

### 3. 进阶：让 MCP 工具消费自定义指令

除了基础信息，`USER.md` 可以包含 **Agent 专属 section**，例如：

```markdown
## Agent Rules
- When creating a PR, always use the template in ~/.github/pull_request_template.md.
- When running `gh` CLI, assume I'm already authenticated.
- The MCP server `project-memory` stores task-level context; prioritize it over generic answers.
```

对应地，自定义 MCP 服务器可以解析这些规则并暴露为工具函数的默认参数。例如，一个 `git_helper` MCP 工具可以读取 `Agent Rules` 里的 PR 模板路径，自动应用。

## 踩坑点

### 1. “一次性加载” vs.“按需查询”

最开始的直觉是把整个 `USER.md` 贴在 system prompt 里，但很快会遇到 token 开销问题。一个 500 行的 `USER.md` 可能占用 1500+ token，每次对话都携带是对上下文窗口的浪费。经验是：**只让 Agent 知道这个文件存在，并保留随时查询的能力**。使用 MCP 文件系统工具按需读取，是更经济的方案。

### 2. 敏感信息泄露

`USER.md` 很可能包含 API key、路径、内部域名等。如果你使用任何自动上传、日志分析或共享对话链接的功能，需要确保这个文件被排除在外。做法：

- 将 `USER.md` 放在 `.gitignore` 中，不在项目仓库里提交。
- 在日志采集工具中过滤该文件路径。
- 如果使用云端 Agent，可以考虑对 `USER.md` 内容做脱敏处理，或使用环境变量替换敏感值（如 `$OPENAI_API_KEY`），让 Agent 在读取时从环境注入。

### 3. 文件不存在时 Agent 行为异常

如果没有 `USER.md`，一些 Agent 可能反复尝试读取，造成循环错误。务必在 system prompt 中定义“文件缺失”时的降级策略，例如：“If USER.md not found, proceed with defaults, but remind the user once.”

### 4. 格式解析脆弱

如果你在 `USER.md` 中写了自定义 markdown 块，Agent 的解析可能会因为 `#` 数量不一致、缩进问题而失败。设计约定比格式解析更重要。建议只使用固定的 header 层级，不依赖复杂嵌套。

## 可复用建议

- **版本管理**：在 `USER.md` 头部加上 `last_updated` 字段，Agent 可以从变更频率判断是否要主动询问“你的习惯改了吗？”
- **模块化**：如果 `USER.md` 变长，可以拆分为 `user_profile.md`、`project_rules.md`，由主文件通过 `@include` 语法引用，但需 MCP 工具支持多文件读取。
- **模板化分发**：团队可以维护一份 `USER.md` 模板仓库，新成员 fork 后按需填写。OpenClaw 环境中通过初始化脚本自动链接到 `~/.openclaw/`。
- **结合记忆系统**：`USER.md` 是静态知识，配合 OpenClaw 的 conversation memory 或专用 MCP memory server（如 `mcp-memory`），可以在对话中更新“临时偏好”，而无需修改文件本身。

## 总结

`USER.md` 不是什么新奇想法，但它把“个性化 Agent 体验”从口头提示变成了工程文件，从而可以被版本化、共享、自动化。在 MCP 生态下，它的价值被放大：任何兼容 MCP 的工具都能复用这份用户定义，而不会被困在单一聊天窗口中。

如果你的 OpenClaw 还没有一份 `USER.md`，可以从最简版本开始——写上名字、系统、项目路径和三条约束。你会发现，Agent 从一个“能干活的陌生人”逐渐变成“了解你工作节奏的队友”。

---

