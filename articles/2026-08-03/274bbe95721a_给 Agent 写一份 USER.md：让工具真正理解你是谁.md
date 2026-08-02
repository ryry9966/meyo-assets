---
title: 给 Agent 写一份 USER.md：让工具真正理解你是谁
feedId: 31404
source: 综合讨论
publishedAt: 2026-08-03
---

# 给 Agent 写一份 USER.md：让工具真正理解你是谁

## 背景

用 AI 写代码、做自动化的人，迟早会面临同一个问题：每次开新会话，都要重新告诉助手“我用 macOS，Homebrew 装了 Python 3.12，习惯用 pytest 而不是 unittest，不喜欢在 Markdown 里放 emoji……”。这些信息既不是项目的一部分，也不是系统提示词的范畴，纯粹是“我这个用户的上下文”。

OpenClaw 这类 Agent 框架强调通过 MCP（Model Context Protocol）和插件系统注入外部上下文，正好提供了一个极好的挂载点：你可以把个人偏好、技术环境、工作习惯写成一个结构化文件，让 Agent 在每次会话中自动加载，而不用每次都从零解释。

## 问题

没有用户级上下文的时候，Agent 的默认行为几乎一定会和你预期相悖：

- 生成 shell 命令时用了 `apt-get`，而你在 macOS 上。
- 建议你用 `pip` 全局安装，而你的项目用 `uv` / `pipenv`。
- 明明项目用 Ruff 格式化，Agent 却一直给你 `black` 的配置片段。
- 你对某个工具链的版本有硬性要求，但 Agent 假设的是最新稳定版。

反复纠正这些差异消耗的 token 和时间，积少成多之后是一笔不小的隐性成本。更关键的是，Agent 如果对你的一贯风格缺乏理解，产出的代码、文档或配置就始终带着“通用模板”的味道，后续手动调整又会引入新的不一致。

## 做法

### 1. 创建全局 USER.md

在你的家目录或固定路径下创建一个 `USER.md`，采用 Markdown 格式，内容按模块划分。典型的模块包括：

- **身份与角色**：你的主要角色（后端 / 全栈 / SRE 等）、习惯使用的语言、工作领域。
- **操作系统与 Shell**：OS、Shell 类型、包管理器、常用 CLI 工具。
- **技术栈与版本**：语言版本管理器、数据库客户端、容器运行时等，要求 Agent 在给出建议前先确认版本的兼容性。
- **代码风格与工具链**：格式化工具、linter、测试框架、类型检查器、Git 工作流偏好。
- **项目结构惯例**：偏好的目录布局、模块命名、测试文件位置。
- **文档与沟通偏好**：语言（中文/英文）、注释风格、是否使用 emoji、日志级别习惯等。
- **约束与禁忌**：绝对不要使用的库、避免的架构模式、安全红线。

示例片段：

```markdown
## 操作系统与工具链
- OS: macOS 14, Apple Silicon
- Shell: zsh, 包管理: Homebrew
- Python: 通过 mise 管理版本，当前默认 3.12
- Node: 20 LTS via volta
- 容器: OrbStack (Docker-compatible)

## Python 代码风格
- Formatter: Ruff，配置统一用 pyproject.toml
- 测试: pytest + pytest-cov，不使用 unittest
- 类型检查: mypy strict 模式
- 依赖管理: uv，生成 requirements.txt 仅作为兼容出口
- 绝对不写 `from module import *`
```

### 2. 让 Agent 加载 USER.md

OpenClaw 或类似的 Agent 框架通常支持在初始化时注入自定义上下文。以 OpenClaw 为例，可以在启动配置中通过 MCP 资源或直接文件读取的方式加载 `USER.md` 全文。如果使用 Claude Code、Continue 等工具，也可以在 “Custom Instructions” 或系统提示词中通过 `@file` 引用。

更工程化的做法是把 `USER.md` 放在一个 MCP 服务器里，作为一个动态资源暴露，这样多个 Agent 可以复用，也能集中更新版本。例如，编写一个简单的 stdio MCP 服务器，提供 `resource://user/profile.md`，Agent 在连接时拉取。

```
MCP Server (filesystem/stdio)
  └─ resource: user://profile.md -> 指向 /Users/you/.config/agent/USER.md
```

Agent 框架配置中增加该 MCP 服务器的连接，即可在每个会话里自动获得用户上下文。

### 3. 与项目级上下文分层配合

除了全局 `USER.md`，项目里通常还有 `CLAUDE.md`、`CONTEXT.md` 或 `AGENTS.md`。为了避免冲突，设定优先级规则：项目级覆盖用户级，用户级覆盖系统默认。在提示词的编排里，把项目文件放在用户文件之后注入，以保证具体约束能生效。

## 踩坑点

1. **Token 消耗膨胀**：`USER.md` 写得过于详尽（比如贴满完整配置文件），会让每次会话的 system prompt 轻松超过几千 token。需要精简，只保留 Agent 真正会做决策所需的信息。可以把详细配置作为可查询的资源，而不是每次都全量注入。

2. **过时信息更危险**：你升级了 Python 3.13，但 `USER.md` 里还写着 3.11，Agent 会基于错误的前提给出建议。解决方案是把 `USER.md` 纳入版本控制（dotfiles 仓库），并在每次环境变更时通过 hook 提醒更新，或者用脚本自动探测环境参数生成 Markdown 片段。

3. **隐私泄露风险**：文件里可能会无意中带上用户名、内部主机名、API key 路径等信息。分享终端截图或把 dotfiles 公开时要格外注意，建议单独维护一个 `USER.public.md` 用于分享，私密版本通过 MCP 在本地加载。

4. **Agent 并不总是遵守**：即便是写得很清楚的偏好，某些模型在长上下文中也可能“忘记”。如果发现 Agent 忽略某条规则，可以把它在 `USER.md` 中加粗或用 `IMPORTANT:` 前缀标记，但这也只是提高概率，不是保证。

## 可复用建议

- **模块化拆写**：把不同关注点拆成独立文件（`preferences/sh.md`, `preferences/python.md`），通过 MCP 根据当前任务动态选择加载哪些部分，减少无关 token。
- **提供示例而非说教**：与其写“请写干净的代码”，不如在 `USER.md` 里给一个代码片段示例，明确你心中的“干净”长什么样。
- **加入自检问题清单**：在文件末尾列几个 Agent 在生成输出前应检查的问题，比如“这个命令在 macOS 上能直接运行吗？”“使用的库是否在项目已有的依赖中？”，这种 checklist 对模型有实际约束力。
- **定期用 Agent 验证**：每隔一段时间，开一个新会话，要求 Agent 根据 `USER.md` 生成一段代码或命令，看它是否偏离预期，然后据此修正文档。

## 总结

`USER.md` 本质上是一份给 Agent 的“用户手册”，它把散落在各次对话里的个人上下文显式化、持久化。如果能工程化地集成到 OpenClaw + MCP 的上下文体系里，它就能像 dotfiles 对你的 shell 一样，默默发挥作用。花半小时认真写好第一版，之后每次避免的重复沟通和认知偏差，都能快速赚回这笔投资。

---

