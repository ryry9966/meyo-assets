---
title: Git 自动化实战：让 OpenClaw Agent 接管代码提交与分支管理
feedId: 36834
source: 综合讨论
publishedAt: 2026-09-10
---

## 背景

OpenClaw 这类 agent 网关接上 MCP 工具或受控 shell 之后，模型就有了操作本地环境的手脚。第一个值得交出去的场景，我选了 Git——不是因为它多难，而是因为高频、规则清晰、几乎一切操作都可回滚，非常适合作为 agent 自动化的试点。

## 问题

团队日常 Git 使用里的机械噪音不少：commit message 随手写（`fix`、`update`、`改完了`）；feature 分支合完不删，两个月后 `git branch` 一屏放不下；changelog 靠发版前人肉翻 log；规范写在 wiki 里但没人看。这些活儿判断量小、重复度高，正好是 agent 该接的部分。

## 做法

我的落地分四层：

1. **工具层**：优先用 MCP 的 git server；退而求其次给一个白名单脚本，只暴露 `status`、`diff`、`log`、`add`、`commit`、`branch -d` 这类低危命令。不要一上来就给完整 shell。
2. **约定层**：在仓库根目录放 `AGENTS.md`（随代码版本化），写清分支命名、Conventional Commits 格式、禁止事项（force push、直接动 main、`git add -A`）。
3. **流程层**：把提交固定成五步——`status` → 分文件喂 `diff` → 生成提交信息 → 人确认 → `commit`。分支清理同理：列出已合并分支 → 提出删除清单 → 确认后执行。任何破坏性操作一律交还给人。
4. **审计层**：agent 触发的每条 git 命令追加写入本地日志文件，出问题能回放。

## 踩坑点

- **只喂 status 不喂 diff**，模型会"脑补"改动内容，写出看似合理实则错误的提交信息。必须喂真实 diff，大改动分文件分批。
- **`git add -A` 是事故之源**。build 产物、本地配置、.env 都可能被一起提交。改成按文件白名单式 add。
- **detached HEAD 状态下照样能提交**，agent 不会主动提醒，提交最后找不到。每步操作前先校验当前分支状态。
- **diff 里的密钥会随上下文进模型 API**。喂给模型前先跑一遍 secret 扫描，这一步不能省。
- **rebase / merge 冲突时 agent 容易进入重试循环**，越解越乱。约定：遇到冲突立即停止，输出冲突文件清单交给人。

## 可复用建议

- 规范写进版本库而不是 system prompt，团队共享、可 review、可演进。
- 一切先 dry-run：让 agent 输出"将要执行的命令 + 理由"，人点头再执行。这一条比任何复杂框架都管用。
- 权限渐进开放：只读 → commit → 分支管理 → 合并，每一档稳定运行一两周再开下一档。
- 让 agent 的提交带上标记（比如 commit trailer `Assisted-by: openclaw`），回溯时一眼可辨。

## 总结

这套东西跑下来的体感是：agent 接管的是规则明确的机械部分，判断和兜底留在人手里。它没有让我"不用管 Git"，而是把每天几十次琐碎操作压缩成了几次确认。自动化程度不取决于模型多聪明，而取决于你敢给它多大的操作面——从小开始，跑稳再扩。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/7781c62bf14f8333.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/ecc4cae7498f7d4d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/5daa6b68513b6156.png)

