---
title: Git 自动化实践：让 AI 助手管提交和分支，但别让它乱来
feedId: 36452
source: 综合讨论
publishedAt: 2026-09-07
---

## 背景

日常开发里，写 commit message、建分支、清理旧分支，这类活儿单个不超过两分钟，但一天重复十几次就很磨人。我们团队从今年开始把这部分交给接入 MCP 的 AI 助手处理，跑了几个月，说说能落地的做法和踩过的坑。

## 问题

直接让 agent 开个 shell 跑 git 命令是不行的。我们试过，它会在错误分支上提交、把 `.env` 一起 stage、甚至自作主张 `push --force`。核心矛盾是：git 的能力面太大，而 agent 对"危险动作"的判断不稳定。自动化要做，但工具面必须收窄。

## 做法

**1. 收窄工具层。** 不给裸 shell，改用 MCP git server 或自写插件，只暴露 `status`、`diff`、`log`、`add`、`commit`、`branch` 这几个安全操作；`push`、`reset --hard`、`rebase` 一律不暴露。每个调用落日志，事后可审计。

**2. 约定文件化。** 提交规范（我们用 Conventional Commits）和分支命名规则写在仓库根目录的约定文件里，agent 每次任务开始先读它。约定跟着仓库走，换 agent 也不用重写 prompt。

**3. 三步提交流。** agent 先执行 `git diff HEAD` 看全量改动 → 按规范生成 message 和目标分支，输出提案 → 人确认后才真正 `add + commit`。生成快、确认慢，出错成本趋近于零。

**4. 分支清理。** 每周让 agent 跑一次清理任务：列出已合并到主干、且超过 14 天没动的分支，生成删除清单，人勾选后批量删。本地和远端分开列，远端删除单独再确认一次。

## 踩坑点

- **message 与改动不符**：只看 staged diff 会漏掉未 add 的文件，务必用 `diff HEAD` 看全量。
- **密钥入库**：agent 分不清 `.env` 该不该提交。我们在工具层加了提交前 secret 扫描，命中直接拒绝并提示，不依赖 agent 自觉。
- **上下文过期**：长会话里 agent 记的仓库状态是旧的。操作前强制重跑 `git status`，别信它的记忆。
- **冲突死循环**：早期版本 agent 遇到合并冲突会反复尝试"修复"，越搞越乱。后来在系统提示里写死：遇冲突一律 abort、汇报、等人工。
- **直接 push**：哪怕只是 fast-forward 也别让 agent 做。push 保持人工触发，或走审批按钮。

## 可复用建议

把这套东西抽象出来就是三条：**最小权限**——工具层决定 agent 能做什么，而不是在 prompt 里求它别做什么；**propose → confirm → execute**——一切写操作先出提案再执行；**约定进仓库**——规范是代码的一部分，不是会话里的临时口头禅。任何想让 agent 碰基础设施的场景，这三条都能直接套。

## 总结

几个月下来的体感是：AI 助手在 Git 上的价值不是"替你思考"，而是把 commit message 的风格统一住、把分支的卫生保持住，把人从重复劳动里捞出来。危险动词留给人类，机械动词交给机器——这条线划清楚，自动化才算稳。欢迎大家在自己的工作流里试一试，踩到新坑欢迎回帖补充。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/47fcf9b1c49a3bbe.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/6edd795d8e3fafaf.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/60c21647b3f39b5d.png)

