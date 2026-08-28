---
title: AI 助手的 USER.md：让 Agent 真正了解你是谁
feedId: 35121
source: 综合讨论
publishedAt: 2026-08-29
---

## 背景

在 OpenClaw 这类 Agent 工作流里，项目级配置越来越常见：`AGENTS.md`、`CLAUDE.md` 会告诉 Agent 当前仓库怎么构建、测试命令是什么、哪些目录不要碰。但有一个空白很少被认真对待：Agent 不了解“你”是谁。

它不知道你的默认 shell、常用编辑器、工作目录习惯、哪些操作需要先确认、哪些依赖已经装好。于是每次开新会话，你都要重复说明；多个 Agent 并行执行时，还会出现彼此不一致的理解。项目文档解决的是“这个项目应该怎么做”，而用户级上下文解决的是“这个人希望怎么做”。

## 问题

自动化任务失败，很多时候不是因为模型能力不够，而是因为 Agent 在错误的环境假设下工作。

几个典型场景：

- 你常用 `fish`，但 Agent 默认生成 `bash` 脚本；
- 你的项目根目录在 `~/work`，Agent 默认去找 `~/projects`；
- 你用 `pnpm`，Agent 按 `npm` 生成命令；
- Agent 自动执行了 `git push --force`，但你的底线是强制推送前必须先确认；
- 多个 MCP server 或子 Agent 读取用户环境时，各自猜测，导致行为不一致。

上下文缺失不只会带来返工，还可能带来实际风险。

## 做法

### 1. 建立用户级 USER.md

建议放在固定路径，例如：

```text
~/.openclaw/USER.md
```

不要在项目里散落多份用户偏好。用户级信息应当只有一个权威来源。

### 2. 内容分块

USER.md 不用写成自传，重点放跨项目、稳态、可执行的信息。我实际使用的结构如下：

```markdown
# Identity
- 称呼：xxx
- 常用语言：中文 / English
- 角色：后端 / 自动化维护

# Environment
- OS: macOS
- Shell: fish
- Editor: Neovim
- Package manager: pnpm

# Workspace
- 主目录：~/work
- 活跃项目：openclaw-runtime, mcp-bridge
- 实验目录：~/lab

# Permissions
- 以下操作必须先输出计划并等待确认：
  - rm -rf / git push --force
  - 数据库 schema 变更
  - 安装全局依赖

# Preferences
- 脚本默认给 fish，除非项目另有要求
- 日志输出使用英文，注释使用中文
- 变更前先给 diff 或计划
```

### 3. 注入方式

OpenClaw 可以在全局配置里把 USER.md 作为基础指令引入。如果支持 MCP resource，也可以把该文件暴露为一个 resource，让 Agent 按需读取，而不是把全文塞进 system prompt。

我目前的做法是：全局配置中仅注入一句“用户级上下文见 `~/.openclaw/USER.md`，初始化任务时先读取”。这样可以减少固定 token 占用。

### 4. 与项目级文档分层

项目级 `AGENTS.md` 依然是最高优先级。USER.md 只描述跨项目稳定偏好，不要重复项目细节。项目文档和用户文档冲突时，应当默认项目文档优先，并在 USER.md 中明确声明这一点。

## 踩坑点

- **不要把秘密写进 USER.md**：API key、token、内部跳板机地址都不要出现。需要引用时只写环境变量名，例如 `$STAGING_TOKEN`，不写值。
- **内容膨胀**：把一次性任务、临时调试记录、当天心情写进去，很快会变成噪音。USER.md 应该像 `.bashrc` 一样克制，只保留长期有效信息。
- **路径跨平台不一致**：`~` 在 Windows 和 macOS/Linux 下展开结果不同。多平台用户建议写环境变量，或明确标注平台分支。
- **项目级冲突**：如果 USER.md 写“默认 pnpm”，项目强制要求 npm，Agent 可能犹豫或选错。需要在文档中写清优先级，例如“当项目级 AGENTS.md 有明确工具链要求时，以项目为准”。
- **隐私泄露**：USER.md 如果被 MCP server 或插件上传到第三方服务，可能暴露你的目录结构、项目名称、工作习惯。不要把公司内部敏感信息放进去，必要时限制 Agent 可访问的用户目录范围。
- **多机不同步**：放 dotfiles 仓库是好习惯，但不同机器环境不同。建议为机器差异留条件块，或用环境变量区分。

## 可复用建议

- 使用 frontmatter 标注更新时间、适用范围，便于脚本解析；
- “必须确认”清单比泛化的“请谨慎操作”更有效，给 Agent 明确可执行的边界；
- 定期清理，建议每月检查一次，删除过期路径和不再使用的偏好；
- 如果 Agent 支持工具调用，可以让它通过工具读取 USER.md，而不是把全文常驻 system prompt；
- 将 USER.md 视为配置代码：有版本、有结构、有边界，而不是随手写的一段说明。

## 总结

USER.md 解决的不是 Agent“会不会”，而是“懂不懂你”。它把重复的自我介绍、环境说明、权限底线沉淀成一份稳态用户上下文，降低每次会话的沟通成本，也减少多 Agent 协作时的理解偏差。

写好 USER.md 的关键不是写得多，而是分层清晰、边界明确、像代码一样持续维护。它不适合承载临时信息，也不应该成为秘密仓库。把它当作 Agent 的“用户配置文件”，自动化才会越来越像你的助手，而不是一个每次都从零开始猜你习惯的陌生人。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/5ac0f134554ae9fd.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/fcb66405af593bc4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/dc9b4c5c2efa7f0c.png)

