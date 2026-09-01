---
title: Git 自动化实战：让 Agent 安全接管提交信息与分支清理
feedId: 35729
source: 综合讨论
publishedAt: 2026-09-02
---

## 背景

用 OpenClaw 跑了一段时间 Agent 之后，最先产生实际收益的不是什么炫酷功能，而是 Git 这类高频、模式化、但人总爱偷懒的操作。本地十几个 `wip`、`fix2`、`tmp-xxx` 分支，提交信息清一色 "update"——这些活 AI 助手其实做得比人稳定。

## 问题

把 Git 交给 Agent，有两个矛盾：

1. 提交信息和分支清理是"低价值高频率"的劳动，人不愿意认真做；
2. Agent 一旦有 shell 权限，`reset --hard`、`push --force`、误删分支这类破坏性操作的风险也随之出现。

所以目标不是"全自动"，而是把 Agent 圈在一个**提议 → 确认 → 执行**的安全循环里。

## 做法

**第一步：封装一个受限的 git skill。** 不要让 Agent 直接用裸 shell 跑 git，而是做一个技能，内置命令白名单：允许 `status / diff / log / branch / commit`，禁止 `reset --hard`、`push --force`、`clean -fd`。白名单写进 skill 定义里，而不是写在提示词里——提示词可能被绕过，白名单才是硬约束。

**第二步：提交助手流程。** 触发后 Agent 依次执行：

- `git status --short` 看变更全貌；
- `git diff --stat` 评估规模，太大就分文件定点看；
- 按 Conventional Commits 规范草拟提交信息，附一段变更摘要；
- 输出建议，等人确认后再 `git add`（精确到文件）和 `git commit`。

确认环节不能省。Agent 对 diff 意图的理解偶尔会跑偏，宁可多按一次回车。

**第三步：分支清理任务。** 用 OpenClaw 的 cron 挂一个周任务：`git branch --merged main` 列出已合并分支，交叉比对远端状态，输出一份"建议删除清单"，人工勾选后统一删除。全程不碰未合并分支。

## 踩坑点

- **`--merged` 不完全可靠**：走 squash merge 的分支不会出现在列表里，只看本地状态会漏判，要结合 `git log --grep` 或远端 PR 状态二次确认。
- **密钥误提交**：Agent 图省事可能 `git add .`。必须在 skill 里强制先跑 `git diff --cached` 审查，维护好 .gitignore，有条件再接一个 secret 扫描步骤。
- **工作目录漂移**：Agent 可能在错误的 worktree 里执行命令。skill 配置里固定 `cwd`，每次执行前用 `git rev-parse --show-toplevel` 校验。
- **上下文爆炸**：几千行 diff 直接塞给模型效果很差，先 `--stat` 再定点看，必要时让它输出文件级摘要。

## 可复用建议

- 一句话概括经验：**权限约束写进工具层，判断逻辑留给模型，最终决定权留给人。**
- 提交信息统一 Conventional Commits，之后 `git log --grep` 才有意义，Agent 的历史行为也可审计。
- 把 Agent 执行过的每条 git 命令追加到日志文件，出问题能回放。
- 先在镜像仓库或沙箱 clone 里跑通整个 skill，再接到真实项目。

## 总结

Git 自动化是 Agent 落地里性价比很高的一类任务：模式清晰、结果可验证、出错可回滚。但也正因为它碰的是版本历史，护栏设计比能力设计更重要。让 Agent 做它擅长的部分——读 diff、起名、列清单——把 add、commit、push、delete 这些落锤动作留在人手里，这套流程就能长期稳定跑下去。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/3437d3fad9cec3dd.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/9e4fa63554070f90.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/5c674ece2d98498c.png)

