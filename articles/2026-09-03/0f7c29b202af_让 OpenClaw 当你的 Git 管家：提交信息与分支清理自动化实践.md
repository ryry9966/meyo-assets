---
title: 让 OpenClaw 当你的 Git 管家：提交信息与分支清理自动化实践
feedId: 35970
source: 综合讨论
publishedAt: 2026-09-03
---

## 背景

个人项目和小团队仓库多了之后，Git 的“杂活”占比会越来越高：写提交信息、同步 main、清理 stale 分支。OpenClaw 的 agent 本身有 shell 执行能力、定时任务和聊天频道推送，天然适合当这个“Git 管家”。最近我把这套流程跑起来了，记录一下实际做法。

## 问题

- 提交信息敷衍：满屏的 `fix`、`update`，一周后自己都看不懂改了什么
- 分支越积越多，`git branch` 一屏放不下，哪些已合并全靠记忆
- 想让 AI 代劳，又怕它一句 `push --force` 把现场炸了

核心矛盾是：既要自动化，又不能把危险操作也自动化。

## 做法

**1. 先划权限，再谈自动化。** 给 agent 单独的工作目录和最小权限密钥，可执行命令收进白名单：`status / diff / log / branch / add / commit / switch`。push 和 reset 一律要求人工确认，force 类操作直接禁止。

**2. 提交信息生成固定成流程。** `git status` → `git diff --staged` → 按 Conventional Commits 草拟 message → 回显 diff stat 让人过目 → commit。diff 超过 400 行时，先让 agent 给 `--stat` 并建议拆分提交，而不是硬吞整段。

**3. 分支体检走定时任务。** 每周跑一次只读脚本：`git branch --merged main` 列出可清理项，找出 14 天无提交的分支，汇总成报告推到聊天频道。只报告，不删除；删除永远等你回一句确认。

**4. 留审计日志。** Agent 执行的每条 git 命令追加写入日志文件，出问题可以回放现场。

## 踩坑点

1. **worktree 漂移**：agent 有次在 detached HEAD 上提交，白干一场。现在每次操作前强制校验 `git rev-parse --abbrev-ref HEAD` 和当前目录。
2. **大 diff 撑爆上下文**：整段 diff 喂进去，后半段改动它就“看不见”了，message 写得像模像样但漏了关键文件。改成 stat 先行、分块摘要。
3. **生成信息千篇一律甚至编造**：要求 message 引用涉及的具体文件名，和 diff 交叉核对，对不上就打回重写。
4. **定时任务与工作区冲突**：体检脚本严格只读；工作区有未提交改动时只提醒，不自作主张 stash。
5. **服务端兜底**：客户端白名单之外，服务端开 `receive.denyNonFastForwards`、保护 main 分支，做双保险。

## 可复用建议

- 把整套东西做成一个 skill：受限 git 脚本 + 固定 prompt 模板，换仓库只改路径
- 白名单和确认规则写在仓库内的配置文件里，跟着代码走、跟着代码审
- 渐进放权：纯只读跑两周 → 开放 commit → 最后才考虑 push
- 一切自动化遵循“先报告后执行”，“人确认”这一步不要省

## 总结

AI 助手擅长的是 Git 里重复、琐碎、容易忘的部分：草拟提交信息、盘点分支、定期汇报。真正危险的决策——push、删分支、改历史——留在链路末端的确认环节。权限收紧、命令可审计、报告先行，这套管家跑了一个月，最直观的变化是：`git branch` 终于一屏能放下了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/dee56218e62f2e54.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/1f9293d372b1a95c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/5c4175db251cf62d.png)

