---
title: 给 AI 助手装上 Git 手刹：可审查的提交与分支自动化
feedId: 35438
source: 综合讨论
publishedAt: 2026-08-31
---

在 OpenClaw-CN 的很多实践里，我们让 agent 跑命令、读文件、调 API，但一碰到 Git 仓库，往往会犹豫：给写权限怕搞乱，不给又没法自动管理提交和分支。本文分享一套偏工程化的做法：不让 AI 直接“裸奔”提交，而是把它接入一个可审查、可回滚的 Git 工作流。

## 背景

日常开发里，大量 Git 操作是低价值重复劳动：整理 diff、写规范 commit message、按需求建 feature/fix 分支、清理过期本地分支。这些任务适合让 AI 助手处理，但直接给 shell 执行 `git add . && git commit` 风险很高。

## 核心问题

AI 做 Git 自动化容易踩中四类问题：

1. 乱写 commit message，风格漂移。
2. 把调试代码、临时文件、密钥或大文件一起 add。
3. 在错误分支或 detached HEAD、rebase 状态下执行写操作。
4. 自动提交误触发 CI、hook 或 push 到远端。

所以关键不是“自动提交”，而是“生成 + 审查 + 执行”三段式。

## 做法

在 OpenClaw 中，我更推荐把 Git 操作拆成只读和写操作两个权限层。只读接口常开：`git status --short`、`git diff --stat`、`git diff -- <file>`、`git branch --show-current`。写操作默认不授权，或者仅授权 `commit`、`branch`、`checkout -b`，禁止 `push --force`、`commit --amend`、`reset --hard`。

一个可复用的流程如下：

1. **只读分析**：让 agent 读取当前分支、变更文件和 diff，输出变更摘要。
2. **生成建议**：agent 根据 staged/unstaged 内容，给出符合规范的 commit message、建议分支名。
3. **人工确认**：在终端或 OpenClaw 会话里确认摘要和命令，不确认不执行。
4. **执行并回传**：执行 `git add` 指定文件、`git commit -m`，然后回传 `git status` 和 `git log -1 --oneline`。
5. **分支管理**：建分支用 `git switch -c <type>/<desc>`，删除本地分支前先列出并确认。

如果使用 MCP git server，可以把写操作做成单独的 tool，默认在 OpenClaw 的权限策略里设为 ask；不要把所有 git 命令塞进一个拥有完整 shell 权限的 tool。

## 踩坑点

我实际搭建时遇到几个典型问题：

- **大 diff 爆上下文**：仓库 diff 超过几千行时，agent 会丢失重点。先限制 `git diff --stat`，再按文件逐个分析，必要时设置最大输出行数。
- **忽略秘密扫描**：即便有 .gitignore，也可能有误 add 的 `.env`、私钥。建议写一个 pre-commit 校验，或在 agent prompt 里显式禁止添加敏感文件模式。
- **commit message 规范漂移**：没有约束时，agent 有时中文有时英文，有时超长。把规范写进 system prompt 或 skill，例如固定 `type(scope): summary`，body 用 bullet。
- **CI 被意外触发**：自动 commit 会触发 push 后的 CI；如果只是整理提交，建议明确是否在 message 中加 `[skip ci]`，不要默认加。
- **rebase/detached HEAD**：执行写操作前必须检查工作树和 HEAD 状态，否则会制造混乱。

## 可复用建议

- 封装成 OpenClaw skill：例如 `/git.smart_commit`，内部先快照 `git status`，再分析、确认、执行。
- 固化为 prompt 片段：commit message 规范、禁止提交的文件模式、需要人工确认的操作列表。
- 保留审计日志：在 skill 里记录时间、用户、分支、变更摘要、执行命令，方便回溯。
- 从小仓库或 feature 分支开始验证，别直接在 main 或多人协作分支上全自动操作。
- 读操作自动，写操作 ask；危险操作直接 deny。

## 总结

Git 自动化的价值不在“替人提交”，而在把整理 diff、起草 message、建分支这些环节标准化。给 AI 助手足够的只读上下文，把写操作留在可确认的边界内，才能真正减少负担又不增加风险。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/39bde03431cdc03b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/db4ba3bc7a99fc34.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/c7235efeea9c0986.png)

