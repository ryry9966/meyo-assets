---
title: Git 自动化：给 OpenClaw 加一层受控的提交与分支助手
feedId: 34392
source: 综合讨论
publishedAt: 2026-08-23
---

## 背景

在 OpenClaw 这类 agent 环境里，AI 已经能读写文件、跑命令、改代码，但 Git 管理往往还是靠人手动完成。常见场景是：改完代码后打开终端，`git add .`，写一个 “update” 或 “fix bug”，直接 push，分支名也叫 `test`、`tmp` 或 `dev`。时间一长，提交历史不可读，回滚困难，协作时也说不清这次改动做了什么。

把 Git 提交和分支交给 AI 助手管理，不是为了“自动写代码”，而是把这些重复但规则明确的操作固化下来，减少低级错误。

## 问题

完全放开 Git 权限很危险。Agent 可能会执行 `push --force`、清空工作区、提交密钥或大文件。所以需要设计一个受控的 Git 自动化流程：允许本地提交、创建分支、生成 commit message；push 和删除操作必须经过人工确认。

## 做法 / 步骤

### 1. 封装受限 Git 工具

在 OpenClaw 中可以用插件或 MCP 工具暴露一组白名单命令。建议不要让模型直接拼 shell。例如定义一个 `git_agent` 工具，内部逻辑：

- 只接受参数：`action`（status / diff / plan / commit / branch / log）
- 内部映射到具体 Git 命令
- 对 `commit` 只允许 `git add` 指定文件 + `git commit -m`
- 对 `branch` 只允许 `git checkout -b` 和 `git switch -c`
- 禁止 `push --force`、`reset --hard`、`clean -fd`

如果已有 MCP git server，可以在其外层加一个策略代理。若使用 shell 插件，至少要在 prompt 中显式声明禁止命令，并配合 OpenClaw 的 permission 或 approval 机制。

### 2. 定义 AI 的 Git 工作流

给 agent 的 system prompt 或 skill 文件写清流程：

- 先运行 `git status --short` 和 `git diff --stat`
- 如果工作区不干净，先判断是否有无关文件
- 从 diff 中提取改动意图，按 Conventional Commits 生成提交信息
- 分支命名：`feat/<topic>`、`fix/<topic>`、`chore/<topic>`
- 提交前输出计划，等待用户确认（dry-run）
- 提交后显示 `git log --oneline -1`

### 3. 典型执行示例

用户说：“帮我把当前修改提交，并建一个 feat/search-filter 分支。”

Agent 会先查看状态，发现修改了 `src/search.py`、`tests/test_search.py`，生成：

```
feat(search): add keyword filter to search API

- add `filter` query param
- update tests
```

然后创建分支并提交。关键点是：先把修改文件列表给用户，确认没有多余文件，再执行。工具层应要求 `commit` 必须携带 `files` 数组，拒绝 `git add -A`。

### 4. 与 MCP / 插件结合

如果 OpenClaw 支持 MCP，可以把 Git 操作拆成 resources（`repo://status`、`repo://diff`）和 tools（`commit_files`、`create_branch`）。resources 只读，模型先读再调用 tools，权限边界更清楚。没有 MCP 时，用本地脚本 `git-agent.sh` 封装也能达到类似效果。

## 踩坑点

- `git add -A` 是事故高发区。临时文件、`.env`、构建产物可能被提交。必须让模型显式列出文件，工具层过滤敏感路径。可以配合 `.gitignore`，但不要完全依赖。
- 大 diff 会撑爆上下文。只传 `git diff --stat` 或按文件截断 diff，超过 200 行先让用户确认是否继续。
- 模型生成的 commit message 可能“脑补”不存在的改动。提交信息生成后，应展示 diff 摘要与 message 的对应关系，便于快速发现错误。
- 多人协作时，AI 创建分支前要检查远端是否已有同名分支。用 `git fetch --prune` 和 `git branch -r` 判断，避免冲突。
- 不要自动 push。至少初期只做本地提交和分支，push 由人完成。等稳定后再把 push 加进白名单，但保留确认步骤。

## 可复用建议

- 用 Conventional Commits 固定 message 结构，减少模型自由发挥。
- 工具层实现 dry-run：先返回将要执行的命令和文件列表，用户回复“确认”后才真正执行。
- 把 Git 自动化做成一个可复用 skill，参数少，职责单一：`plan`、`commit`、`branch`。避免把一堆功能塞进一个 prompt。
- 让 AI 在提交前自动跑相关测试或至少 `git diff --check` 检查空白错误。
- 保留人工 review 接口：所有提交在 push 前可被 `git reset --soft HEAD~1` 撤销，不算大问题。

## 总结

Git 自动化不是让 AI 替代你理解代码，而是把“查看改动、写规范提交、建分支”这些机械步骤交给它。工程化的关键是权限边界和确认机制，而不是有多智能。先让 Agent 管好本地提交和分支，再逐步放开 push 和 PR 描述，比一步到位安全得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/8bd5d04642d86a09.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/c436df14586f5f1b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/449e894ac20b4890.png)

