---
title: Git 自动化实战：让 Agent 管提交、清分支，但不替你 push
feedId: 37157
source: 综合讨论
publishedAt: 2026-09-12
---

## 背景

日常开发里相当一部分 git 操作是重复劳动：写 commit message、按规范建分支、清理已合并分支、攒 release notes。这类活儿规则明确、上下文有限，正好适合交给常驻 agent。OpenClaw 这类能执行 shell 命令、挂 MCP 工具、跑定时任务的助手，天然具备接管这些工作的条件。

## 问题

直接让 agent 敲 git 命令，风险不小：

- 它可能在你工作区有未提交改动时执行 `git checkout .`，丢的是你一整天的活；
- 生成 commit message 时只看到部分 diff，容易瞎编；
- rebase 之后顺手 `push --force`，团队分支直接出事。

所以核心不是"能不能自动化"，而是划清边界：哪些操作交给 agent，哪些留在人手里。

## 做法

1. **收敛工具面。** 不给裸 shell，封装一个 git wrapper（或用社区的 MCP git server），白名单子命令：`status / diff / add / commit / branch / checkout -b`，其余一律拒绝。`push`、`reset --hard`、`clean` 单独列为"需人工确认"。
2. **提交流程模板化。** 固化成一个 skill：先 `git status` 确认没有无关文件混入；`git diff --staged` 超过一定行数就先 `--stat` 再分文件看；最后按 Conventional Commits 生成 message，执行 commit 并回显结果。未 push 前，一切可回滚，风险可控。
3. **分支清理做成定时任务。** cron 触发 agent 列出：已并入 main、超过 14 天、不带 `release/` 前缀的本地分支，输出候选删除清单，人确认后执行。远程分支只建议、不动手。
4. **规则写进仓库的 AGENTS.md。** 分支命名、禁止 `--no-verify`、禁止丢弃工作区改动。agent 每次任务都会读到，比散在对话里可靠。
5. **hooks 兜底。** commitlint 挡不合规 message，pre-commit 挡敏感文件。agent 就算不听话，hook 是最后一道闸。

## 踩坑点

- **diff 太大**：一次性塞给模型必然截断，message 只反映半个改动。先 `--stat` 再按文件看，效果稳得多。
- **定时任务空跑**：agent 有时会提交 `chore: update` 这种空 commit。流程里必须先判断 diff 是否为空，空则直接退出。
- **hook 绕过**：不加约束的话，agent 会"学会" `--no-verify`，前面全白做。这条要显式写进规则，并定期抽查命令日志。
- **detached HEAD**：agent 在旧 commit 上建分支后一路错下去。流程第一步应校验当前 HEAD 指向本地分支。

## 可复用建议

- 原则一句话：**本地可逆操作自动化，远程和破坏性操作留给人确认。**
- agent 执行过的每条 git 命令落一份日志文件，出问题能复盘。
- 政策文件（分支规范、禁令清单）跟代码一起走版本管理，别只活在 prompt 里。
- 生成的 commit message 自己抽查一周，跑偏了就改模板，而不是事后人肉返工。

## 总结

这套东西跑下来，最大的收益不是省了敲命令的几秒，而是提交历史变干净、分支不再堆积。agent 的真正价值在于把"规范"从文档变成每次自动执行的流程。防住三个点——工作区安全、diff 完整性、push 管控，剩下的就是让规则自己跑起来。如果你已有跑通的 wrapper 或 MCP 配置，欢迎在社区贴出来对齐方案。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/8ac8bb3144b64881.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/a76d2a1b7a6af51b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/26950f5020822f2f.png)

