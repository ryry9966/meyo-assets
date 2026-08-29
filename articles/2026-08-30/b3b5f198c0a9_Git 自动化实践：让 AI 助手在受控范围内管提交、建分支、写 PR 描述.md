---
title: Git 自动化实践：让 AI 助手在受控范围内管提交、建分支、写 PR 描述
feedId: 35290
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景

在 OpenClaw 这类 agent 自动化环境里，Git 是高频操作：提交代码、切分支、写 PR 描述、整理变更记录。这些事情本身不复杂，但重复、琐碎，且容易因命名不规范、commit message 随意而让仓库历史变得难读。把一部分 Git 操作交给 AI 助手，可以省掉很多机械劳动，但前提是必须把风险边界划清楚。

## 问题

直接让 AI 全权操作 Git 仓库，常见事故包括：

- 误执行 `git reset --hard`、`git clean -fd`，把未提交的改动清掉；
- 在错误的分支上提交，或者顺手 `push --force`；
- 大仓库 `git diff` 过长，上下文爆掉，AI 生成的提交摘要失真；
- SSH 凭证、凭据助手未继承到 agent 运行环境，导致 push 失败；
- 多个 agent 同时操作同一仓库，出现 index.lock 冲突。

所以，要做的不是“让 AI 管 Git”，而是“让 AI 在受控范围内执行 Git 操作”。

## 做法/步骤

### 1. 接入 Git MCP Server

在 OpenClaw 中注册一个 Git MCP Server。可以选择只读优先的工具集，例如先暴露：

- `git status`
- `git diff --staged`
- `git diff -- <path>`
- `git log --oneline -20`
- `git branch --list`

写操作默认不开放，或者用工具权限分级单独配置。

### 2. 设置工作目录白名单

限制 Git MCP Server 的工作目录，只允许在指定仓库路径下执行。这样可以避免 agent 在错误的目录里初始化、提交或删除内容。

### 3. 给 Agent 加系统级规则

在 system prompt 或插件配置里写清楚：

- 禁止执行 `git reset --hard`、`git clean -fd`、`git push --force`；
- 任何写操作前必须先输出计划，等待用户确认；
- commit message 遵循 Conventional Commits；
- 分支名统一使用 `feat/`、`fix/`、`chore/` 前缀，并做 slug 化。

### 4. 典型工作流

用户说：“看看当前改动，帮我提交一下。”

Agent 的实际步骤：

1. 调用 `git status` 和 `git diff`；
2. 分析改动内容，生成提交信息，例如 `fix(auth): correct token refresh interval`；
3. 展示计划：`git add -A && git commit -m "..."`；
4. 用户回复确认后，才执行 commit。

分支创建同理。根据任务类型生成 `feat/openclaw-git-mcp` 这类分支，先检查是否已存在，再执行 `git checkout -b`。

### 5. 配合 Hook 做最后防线

在仓库里配置 pre-commit hook 跑 lint 或测试，避免 AI 提交不完整代码。也可以设置 `GIT_TERMINAL_PROMPT=0`，防止交互式提示让 agent 卡住。

## 踩坑点

- **危险命令黑名单**：`reset --hard`、`clean -fd`、`push --force`、`rebase --onto` 这类命令最好直接禁止，连确认机会都不给。
- **上下文过长**：大仓库 `git diff` 可能几万行。先让 agent 看 `git diff --stat`，或者限制 `git diff -- path1 path2`，按文件逐个处理。
- **认证问题**：MCP Server 运行环境不一定继承 SSH agent 或 Git credential helper。建议显式配置认证，但 token 不要写进日志或配置明文。
- **并发冲突**：如果多个 agent 可能操作同一仓库，限制同一时间只有一个写任务，或使用文件锁。
- **非法分支名**：AI 可能生成带空格或特殊字符的分支名。统一做 slug 化，限制长度和字符集。
- **交互式命令卡住**：设置非交互环境变量，避免 `git commit` 打开编辑器或询问。

## 可复用建议

1. **只读优先，写操作白名单**：先开放 `status/diff/log`，再按需开放 `commit/branch/checkout`。
2. **所有写操作默认 dry-run**：先展示将执行的命令，再执行。
3. **强制人工确认**：commit 和 push 必须等待用户明确确认。
4. **日志审计**：记录所有 Git 命令及结果，便于回溯。
5. **封装成 OpenClaw 插件**：把 Git MCP 配置、规则提示、常用工作流打包，形成一个可复用的 Git 助手插件。

## 总结

Git 自动化不是让 AI 完全托管仓库，而是把重复、低风险的部分交给 agent，把高风险操作留在人工确认环节。工程化的关键不是“多智能”，而是权限边界清晰、行为可审计、失败可恢复。这样，AI 助手才能真正成为 Git 工作流里的可靠协作者，而不是一个会乱按回车的不确定因素。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/2134cfe2cbd3e0a4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/044065c5bc834361.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/fc379e1ed7bb99d4.png)

