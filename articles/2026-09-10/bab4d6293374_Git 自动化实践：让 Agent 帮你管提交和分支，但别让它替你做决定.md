---
title: Git 自动化实践：让 Agent 帮你管提交和分支，但别让它替你做决定
feedId: 36864
source: 综合讨论
publishedAt: 2026-09-10
---

## 背景

最近把日常 Git 操作逐步交给 OpenClaw 的 agent 处理：写提交信息、建分支、清理已合并分支。跑了一个多月，说说真实体验——确实省时间，但前提是把权限和流程框死。

## 问题

痛点很具体：

1. 提交信息质量不稳定。赶时间手写经常是 `fix`、`update` 这种无效信息，事后翻 log 全靠猜。
2. 实验性改动直接在 main 上改，想拆分支时已经晚了。
3. 长期分支堆积，`git branch` 一屏放不下，哪些能删要逐个查。
4. 跨多文件的改动，该拆几个 commit 全凭心情。

这些事不难，但琐碎，恰好是 agent 适合接手的类型。

## 做法

核心思路：**agent 只做"读 + 提案"，写操作走人工确认。**

**第一步，配受限工具。** 给 agent 一个受限 shell（或 MCP git server），凭证走 credential helper，禁止在命令行里出现 token。

**第二步，系统提示词里写死规则：**

- 提交前必须先跑 `git status` 和 `git diff --staged`，基于真实 diff 写 message，禁止凭记忆总结
- 禁止 `git push -f`，禁止直接 push 到 main
- 新任务先建分支，命名跟仓库规范走（`feat/xxx`、`fix/xxx`）
- 提交信息用 Conventional Commits，body 写清楚 why，不只是 what

**第三步，固化高频工作流。** 我的"提交"流程是：agent 输出「当前分支 / 暂存内容摘要 / 拆分方案 / 每个 commit 的 message」，我确认后才执行。"清理"流程是：agent 列出已合并进主干的本地分支，我勾选后它执行删除。

**第四步，钩子兜底。** pre-commit 跑 lint 和敏感文件检查，agent 提交也要过这道门。

## 踩坑点

- **`git add -A` 是重灾区。** agent 图省事会把 `.env`、构建产物一起加进来。后来直接在提示词里禁用 `-A`，要求逐文件指定，配合 `.gitignore` 双保险。
- **信息幻觉。** agent 有时会"脑补"改动内容写 message，和实际 diff 对不上。必须强制它引用 diff 输出再写。
- **交互式命令跑不了。** `git rebase -i` 在非交互 shell 里直接挂。改用 `git rebase --onto` 或逐条 cherry-pick 替代。
- **自提交循环。** agent 改代码改到一半自己 commit，历史全是碎块。规定它只在明确收到"提交"指令时才动手。

## 可复用建议

1. **权限分层**：`status`/`diff`/`log` 放开，`commit` 需确认，`push`/`reset --hard`/`clean` 一律人工。
2. **仓库里放 `.gitmessage` 模板**，人和 agent 用同一套规范，别搞双标。
3. **记录 agent 执行过的 git 命令日志**，出问题能回溯，复盘也有依据。
4. **从零风险任务起步**：生成 message、列分支、写 changelog，跑顺了再逐步放开写权限。

## 总结

Git 自动化的价值不在"全自动"，而在把机械劳动挪走、把判断留给人。Agent 写的 message 比我赶时间时随手写的强，分支也不再堆积——但它必须是戴着镣铐的协作者。权限最小化 + 提案确认制，是目前验证下来最稳的组合。有兴趣的同学可以先从"提交信息生成"这一件事试起，一周后你自己会知道该不该往下走。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/6c57b5a699482245.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/d3e80959e948c276.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/92d040a22c0a7b6f.png)

