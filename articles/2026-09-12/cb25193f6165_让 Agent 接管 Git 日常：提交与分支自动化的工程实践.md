---
title: 让 Agent 接管 Git 日常：提交与分支自动化的工程实践
feedId: 37200
source: 综合讨论
publishedAt: 2026-09-12
---

## 背景

OpenClaw 这类常驻 agent 的价值，不在于替你写代码，而在于吃掉那些机械、高频、容易拖延的操作。Git 恰好是重灾区：写 commit message、拆提交、清理分支、回忆“昨天改了啥”。这些事单件不超过两分钟，但一天累积下来非常干扰状态。

我的做法是把 agent 接入项目仓库，让它承担 Git 的“日常运维”，人只保留判断权。

## 问题

实际痛点有三个：

1. 提交信息退化成 `fix`、`update`，两周后自己都看不懂；
2. 分支只增不减，`git branch` 一屏放不下，哪条还能删全靠猜；
3. 切分支前忘 stash、提交前忘检查 `.env`，出过两次险情。

## 做法

**第一步：限定权限边界。** agent 通过 shell 工具操作 git，我把命令做了三级划分：只读命令（`status` / `diff` / `log` / `branch -a`）直接放行；常规写操作（`add` / `commit` / `switch` / 新建分支）允许，但要求先汇报 diff 摘要；危险操作（`push --force`、`reset --hard`、`clean`、对公共分支 rebase）硬性拒绝，只提示我手动执行。

**第二步：把约定写进项目指令文件。** 在仓库根目录的 agent 指令文件里固化规范：提交遵循 Conventional Commits、分支用 `feat/` `fix/` 前缀、main 禁止直接提交、涉及 `.env` 和密钥文件的变更必须显式确认。约定进了版本库，agent 每次会话都能读到，团队也能 review。

**第三步：定义三类高频任务。**
- 提交整理：“把当前改动按逻辑拆成提交，起草符合规范的 message"——agent 先跑 `git diff` 分组，再逐个 stage 提交；
- 分支巡检：“列出已合入 main 且超过两周的本地分支”，确认后批量删除，远程只出报告不动手；
- 每日摘要：用 cron/heartbeat 触发，早上推送一份“未提交变更 + 落后远程的分支”简报。

**第四步：把 pre-commit 当最后防线。** lint-staged 和 gitleaks 照常挂在 hook 上。agent 再“聪明”也可能漏检密钥，hook 是确定性的，这道闸不能省。

## 踩坑点

- **`git add -A` 是重灾区。** 早期让它“提交所有改动"，结果把 `.env.local` 一起 staged 了。后来明确禁用 `-A`，必须按文件路径添加。
- **脏工作区切分支。** 有一次 agent 没检查 status 就 `switch`，未提交改动被带去了别的分支。现在规定：任何切换前先跑 `git status`，非干净状态必须先询问。
- **rebase 开着 PR 的分支。** 本地 rebase 后远端分叉，同事拉取时历史错乱。公共分支的 rebase 已列入硬拒绝清单。
- **多仓库串台。** 在错误目录执行清理命令的风险真实存在。我的习惯是任务开头让它复述 `git rev-parse --show-toplevel`，确认仓库根再动手。

## 可复用建议

1. 先只读、后写入：让 agent 跑两周"日报模式"再开放写权限，观察它对 diff 的理解是否靠谱；
2. 约定文件化：规范放仓库里而不是聊天里，可 review、可回滚；
3. 动词收敛：把任务收敛成“整理 / 巡检 / 摘要”等少数明确动词，显著降低误解率；
4. 留审计线索：agent 会话记录 + `git reflog` 就是完整的操作日志，出问题能回溯。

## 总结

这套组合跑了一个多月，commit message 质量肉眼可见地回升，本地分支稳定在个位数。核心经验一句话：agent 负责消除摩擦，约定和 hook 负责兜底，危险操作永远留在人手里。自动化程度可以逐步放开，但权限边界一开始就要画死。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/2390da7e089c0e92.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/a7bb4631fda8eaba.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/8fcd9df7813fa0f1.png)

