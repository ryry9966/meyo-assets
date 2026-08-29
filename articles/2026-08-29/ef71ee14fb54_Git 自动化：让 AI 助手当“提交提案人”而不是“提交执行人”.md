---
title: Git 自动化：让 AI 助手当“提交提案人”而不是“提交执行人”
feedId: 35210
source: 综合讨论
publishedAt: 2026-08-29
---

## 背景

在 OpenClaw 的 Agent 工作流里，代码提交和分支管理是典型的“高频低决策”操作。让 AI 助手处理这些操作，能减少开发者频繁切换终端、整理 commit message、来回切分支的负担。但 git 本身不可逆，直接给模型一个裸 shell 执行 git 命令，风险远大于收益。

更合理的方式，是通过 MCP、插件或自定义工具把 git 能力封装成受限操作，让 AI 只做“提交提案人”，而不是“提交执行人”。

## 问题

直接让模型执行 `git add . && git commit`，通常会遇到几类问题：

- commit message 笼统，如 “update code”；
- 把 `.env`、密钥文件、`node_modules` 等一并提交；
- 大 diff 一锅提交，后续难以回滚；
- 多个 agent 同时操作同一仓库，产生 `index.lock`；
- 更危险的是 `push --force`、`reset --hard`、`clean -fd` 等不可逆命令被误触发。

核心不是“不让 AI 用 git”，而是把它限制在“提案—确认—执行”的链路里。

## 做法/步骤

### 1. 封装 Git 工具，不要给裸 shell

通过 Git MCP server 或 OpenClaw 插件暴露白名单命令。例如只允许：

- `status`
- `diff`
- `log`
- `branch`
- `add`
- `commit`

对 `push --force`、`reset --hard`、`clean -fd`、`rebase` 等命令默认拒绝。工具必须绑定工作目录，不能访问任意路径。

### 2. 强制先读后写

在 prompt 中要求 AI：

- 先执行 `git status --short` 和 `git diff --stat`；
- 再逐文件查看 diff；
- 未得到人工确认前，不准 `add` 或 `commit`。

### 3. 统一提交信息

使用 Conventional Commits 模板：

```text
feat: add branch auto-creation flow
fix: prevent env files from being staged
chore: update git hooks
```

要求 AI 说明变更动机，而不是简单罗列文件。

### 4. 分支规则

只允许从 `main` 创建 `feature/xxx`、`fix/xxx` 分支，不直接在 `main` 上提交。创建分支前先检查本地是否有未提交变更。

### 5. 接入检查钩子

在 pre-commit 阶段跑 lint、test 和 gitleaks/git-secrets。检查不通过时，AI 不能继续提交。

一个最小 prompt 模板：

```text
你是代码提交助手。
先运行 status 和 diff stat，再展示拟提交文件列表。
只提交我明确确认的文件。
禁止提交 .env、*.pem、*.key、*secret*。
提交信息使用 conventional commits，不超过 72 字符。
默认不执行 push，不执行 reset --hard 或 clean -fd。
```

### 6. 生成 PR 描述

基于 `main..HEAD` 的 log 和 diff，生成背景、变更点、测试情况、风险四段描述，方便人工复核后合并。

## 踩坑点

- **不要给 AI 完整 shell 权限**。即使工具封装，也要显式黑名单不可逆命令。
- **敏感文件边界**。AI 不了解哪些文件不能提交，需要在 prompt 中明确列出，并用 gitleaks 在 pre-commit 阶段兜底。
- **大 diff 一次提交**。要求 AI 按主题分多次提交，先提方案再执行。
- **并发 git 操作**。多个 agent 同时操作同一仓库容易出现 `index.lock`。可以加文件锁，或限制单仓库单 agent。
- **未提交变更时切换分支**。强制先查看 `status`，有变更时先提交或 stash。
- **自动 push 事故**。默认关闭 push，只生成 commit 和 PR 描述，由人完成 push/merge。

## 可复用建议

将 git 规则写入仓库的 `AGENTS.md` 或 OpenClaw 知识库，保证每个 agent 都读取同一份规则。提供 dry-run 模式：AI 先输出完整执行计划，经确认后才调用写操作。所有 AI 发起的 git 命令记录到审计日志。敏感仓库给 agent 使用单独 clone，避免污染主工作区。

最后，把 AI 定位为提案人：创建分支、整理提交、写 PR 描述可以交给它，push 和 merge 留给人。

## 总结

Git 自动化不是让 AI 替代人的判断，而是把重复、可描述、可审计的部分交给 AI。通过 MCP 工具白名单、强制先读后写、提交模板和检查钩子，能在 OpenClaw 环境里实现实用且相对安全的代码提交与分支管理。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/bf3a113f26e2deea.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/684a6024a4864bd2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/de4918cdd759e5f4.png)

