---
title: 让 Agent 帮你管 Git：提交与分支自动化的落地实践
feedId: 36752
source: 综合讨论
publishedAt: 2026-09-09
---

## 背景

在 OpenClaw 的典型用法里，Agent 已经能读写文件、跑命令，代码改动大量交给 AI 完成。但 git 操作往往还停在手动：改完自己 diff、自己写 commit message、自己切分支。这一段恰好是流程中最机械、最容易不规范的环节，也最适合交回给 Agent。

## 问题

实际跑起来后，常见三类毛病：

1. **提交粒度失控**：Agent 一次改十个文件，一个 `git add .` 全塞进一个 commit，回滚和 review 都困难。
2. **message 与 diff 脱节**：提交信息写得头头是道，实际改动对不上。
3. **分支纪律缺失**：直接在 main 上改，或者实验分支开了不收尾。

## 做法

我目前的方案是「MCP git 工具 + 提示词约定 + hook 兜底」三层结构：

**1. 最小权限的 git 工具**
通过 MCP 暴露 git 能力，只放行 `status / diff / log / add / commit / branch / switch / merge --no-ff` 这类安全子命令。`push`、`reset --hard`、`push --force` 一律不允许，push 由人触发或走单独审批。

**2. 在技能文件里固化工作流**
给 Agent 的技能/系统提示里写死规则：

- 接到任务先 `git switch -c feat/xxx` 或 `fix/xxx`，禁止直接改 main；
- 提交前必须先跑 `git diff`，message 从真实 diff 归纳，遵循 Conventional Commits；
- 一次任务一个分支，收尾时列出待合并分支，交给人 review。

**3. hook 做最后防线**
提示词会被忽略，hook 不会。`commit-msg` 用 commitlint 卡格式；`pre-commit` 跑 lint 和敏感文件扫描（拦 `.env`、密钥）；CI 侧保护 main 分支。

整条流程串起来：领任务 → 建分支 → 改代码 → 自查 diff → 分文件 stage → commit → 汇报等待人工 push/合并。

## 踩坑点

- **`git add .` 是重灾区**：Agent 很爱用。除 .gitignore 外，我在提示词里明确要求逐文件 stage，并在提交前打印「文件清单 + message」，确认后再执行。
- **message 幻觉**：Agent 会声称"修复了全部测试"，实际只改了两行。要求 message 只描述 diff 中可见的变更，不写意图性结论。
- **冲突自动合并**：让 Agent 自动解冲突风险很高，可能悄悄丢代码。现在规定遇到冲突必须停下，把冲突块原样展示给人。
- **detached HEAD**：Agent 偶尔 checkout 到某个 commit 后迷路。加了一条自检：每次 git 操作前先 `git status`，发现 detached HEAD 先回到任务分支。
- **WIP 噪音**：一个任务十几个碎 commit。让 Agent 自主 rebase 不现实，改为约定：同分支后续 commit 用 fixup 语义描述，合并前人工 squash。

## 可复用建议

- **约定放提示词，红线放 hook，权限放工具层**——三层各管一段，别指望任何一层单独兜底。
- 给 Agent 一个独立的 worktree/clone，搞砸了直接删掉重来，主仓库永远干净。
- 把 Agent 执行的 git 命令记日志，出问题能回放。
- 提交前先输出 dry-run（文件清单 + message），比事后回滚便宜得多。

## 总结

让 AI 管 Git 的核心不是信任它，而是把规范做成它绕不过去的结构：提示词表达意图，hook 强制约束，权限限制爆炸半径。这套三层结构搭好后，日常提交和分支管理基本可以放手，人只保留 push 和合并两个决策点——这大概是目前自动化程度和安全之间的最佳平衡。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/e1988b158eae9726.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/59ee76126011d4d7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/6571f8732fa1aa15.png)

