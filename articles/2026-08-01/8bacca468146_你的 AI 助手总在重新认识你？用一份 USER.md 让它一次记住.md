---
title: 你的 AI 助手总在重新认识你？用一份 USER.md 让它一次记住
feedId: 31173
source: 综合讨论
publishedAt: 2026-08-01
---

## 背景：每次聊都要重新自我介绍的 AI

使用 OpenClaw 或任何支持持久化助理的 Agent 框架时，一个很常见的痛点会被反复触发：**每次打开新会话，你都要重新告诉 AI 你是谁、你的项目在哪儿、你习惯怎么工作**。哪怕只是让助手帮忙改个代码、生成一条 SQL，也得先喂一波上下文。

有些框架会在项目根目录放一个 `.openclaw/context.md`，有些则通过自定义 system prompt 注入信息。但这些机制要么绑定某个项目，要么维护成本很高，跨项目、跨会话的**用户级记忆**始终缺位。

解决这个问题的工程化手段之一，是给 AI 助手一份 **USER.md**。

## USER.md 是什么

可以把它理解成一个**持久化的用户自述文件**，由助手在工作开始时自动读取。文件里记录的不是一次性的任务描述，而是你作为用户的相对稳定的信息：

- 身份与目标（你做什么的，长期在捣鼓什么事）
- 工作环境（常驻终端、笔记工具、项目目录结构）
- 习惯与偏好（代码风格、命名规范、要不要单元测试）
- 常用指令简写（例如 `@deploy` 对应的完整流程）
- 已知的坑和注意事项（某些服务必须走内网、某数据库只读）

这些信息一旦被 Agent 预先加载，就不再占用每次提问的 token 额度，还能减少你的重复解释成本。

## 在 OpenClaw 及其生态中的落地

OpenClaw 的设计里，助理的初始化阶段可以加载一个或多个 **user profile 文件**。例如，可以在 `~/.config/openclaw/user.md` 下放置一份 Markdown 文件，并在助理配置中引用：

```yaml
assistants:
  my-agent:
    profile: "~/.config/openclaw/user.md"
```

或者，利用 OpenClaw 的 `include` 指令，将 USER.md 嵌入到某个技能的 system prompt 前面。MCP 工具链也同样适用——在 MCP Server 启动时指定 `--user-profile`，把这份信息作为全局上下文传给所有工具调用。

这样，无论是本地助手、Code Agent 还是通过 MCP 连接的远程代理，都能够读到同一份“关于你”的基础数据。

## 如何搭建一份可维护的 USER.md

我踩过几次坑后，总结出一个比较稳定的文件结构。总共控制在 **800–1200 词**左右，既够用，又不会让模型处理它时吃掉太多 token。

**实用模板：**

```markdown
# About Me

## Identity
- Role:后端工程师，偏基础设施
- Active projects: 内部运维平台、CI 流程改造
- Tools I always use: neovim, tmux, gh, kubectl, terraform

## Environment
- OS: macOS / Ubuntu server
- Home dir: /Users/leon
- Dotfiles managed by chezmoi
- Notes stored in Obsidian vault: ~/notes

## Preferences
- Always write commits in conventional commits format
- Use double quotes in SQL, single quotes in shell
- Never generate emoji unless I explicitly ask
- For Python scripts, use argparse; avoid click unless it's a CLI tool
- When in doubt, ask to clarify before coding

## Projects & Layout
- ~/workspace/infra: Terraform + Ansible, keep state in S3 backend
- ~/workspace/api: Go services, uses DDD-lite layout
- ~/workspace/scripts: python/bash tools, symlinked to /usr/local/bin

## Shortcuts
- `@lint`: check changed files with ruff+golangci-lint, auto-fix safe issues
- `@deploy-staging`: trigger staging deploy workflow via gh, post result to Slack

## Known Issues
- staging database is read-only for non-admin accounts
- VPN is required to reach internal registry
```

这种结构的好处是：每一个区块都是一个可独立更新的模块。换项目时改一改 `Projects` 部分就好，不需要重写整个文件。

## 踩坑记录

### 1. 文件越长，有效信息密度越低

我曾经把 CI 的完整配置指南也塞了进去，导致文件飙升到 5000 多词。模型开始频繁忽略后面的短指令，出现"说了等于没说"的尴尬。后来我把详细的操作手册拆成单独的 `howto-*.md`，只在 USER.md 里保留**索引和路径**，让 Agent 按需检索。

### 2. 敏感信息很容易被无差别注射

别在 USER.md 里放 token、密码、私钥。哪怕觉得对话历史只有自己看得见，Agent 也可能在生成命令、日志或共享会话时把这些信息带出去。如果需要管理凭证，建议用专门的 `env` 注入或密钥管理工具，USER.md 里最多给个路径提示，比如 “API key stored in 1Password item ‘infra-staging’”。

### 3. 文件版本与真实环境不同步

换电脑、换工作区或者更新工具链后，USER.md 经常会被忘记同步。我的做法是把它纳入 dotfiles 仓库，并用符号链接指向 OpenClaw 的配置路径。每次 `chezmoi update` 之后，助手拿到的就一定是当前最新版。

### 4. 团队协作时记得区分个人与组级配置

如果你帮团队搭建 Agent 工作流，需要把“公司通用配置”和“个人 USER.md”拆开。组级配置写在项目的 `.openclaw/team.md` 里，个人偏好放在自己的 profile 文件中，避免互相污染。

## 可复用建议

- **渐进式搭建**：不用一次写满所有内容。先填 Identity 和 Preferences，用几天感觉缺啥补啥。
- **加个 “Last updated” 字段**：让助手知道信息新鲜度，当发现日期过旧时可以提醒你更新。
- **用片段而非整文件注入**：某些场景（比如只做代码审查的 Agent）不需要读取你的全部信息，可以在技能里用 `include:user.md#Preferences` 这种形式只加载相关片段。
- **结合本地检索**：把 USER.md 和你的笔记库打通。例如 Obsidian 里的一个页面 `AI Profile`，通过 OpenClaw 的 file system tool 加载，这样编辑和 Agent 读取可以在同一个界面完成。
- **定期审查**：每两周或每个迭代结束时扫一眼 USER.md，去掉不再使用的项目路径和指令，保持文件“肌肉感”。

## 总结

为 AI 助手准备一份 USER.md，本质上是**把重复的自我介绍工作沉淀为基础设施**。在 OpenClaw 这类 Agent 框架里，这份文件可以让你的助理从一个“每次都要问你是谁”的陌生工具，变成一个知道你工作方式的协作伙伴。

实施成本极低——建一个几百词的 Markdown 文件、挂到配置里，长期收益却源源不断。它不是一个一次性配置，而是一份需要持续修剪的活文档，但这也正是它最“工程化”的地方：**用管理代码的方式管理你的上下文，让 AI 真正以你为中心工作**。

---

