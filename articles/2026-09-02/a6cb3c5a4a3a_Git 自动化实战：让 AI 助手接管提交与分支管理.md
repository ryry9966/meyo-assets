---
title: Git 自动化实战：让 AI 助手接管提交与分支管理
feedId: 35837
source: 综合讨论
publishedAt: 2026-09-02
---

# Git 自动化实战：让 AI 助手接管提交与分支管理

## 背景

写代码之外，Git 上有一堆高频但低创造性的动作：写 commit message、切分支、清理本地堆积的已合并分支、提交前确认没把 `.env` 带进去。每件事只花一两分钟，但很打断心流。OpenClaw 的 agent 能力加上 MCP 的 git 工具，可以把这些操作收编进一套可审计的流程。这篇帖记录我落地两周的做法和踩坑。

## 问题

三个真实痛点：

1. commit message 质量不稳定，"fix"、"update" 满天飞，回溯时完全没法用；
2. 经常忘切分支，改完才发现写在了 main 上；
3. 本地四十多个分支，一半已合并，不敢乱删。

## 做法

**第一步：接入 git 工具。** 给 OpenClaw 配上 MCP git server（或 shell 执行器），把工作目录固定为常用仓库路径，避免 agent 在错误目录里执行命令。

**第二步：写一个 git skill，固化流程。** 核心是让 agent 按固定顺序操作：

```
git status → git diff（含 --staged）→ 按规范生成 message → 回显确认 → commit
```

message 强制 Conventional Commits 格式，且只允许描述 diff 里实际存在的变更，不许脑补。

**第三步：分支规则写进 skill。** 新功能一律 `feature/xxx`，修 bug 用 `fix/xxx`。agent 动手改文件前先检查当前分支，落在 main 上就先提醒再切分支。每周清理一次：跑 `git branch --merged main` 列出候选清单，等确认后再删。

**第四步：设护栏（最重要）。**

- 禁止 `push --force`，禁止直接改 main；
- push 默认需要我在对话里确认；
- `git add` 前检查暂存区，发现 `.env`、密钥类文件直接拦截；
- rebase、`reset --hard` 这类改历史的操作，agent 只能给建议命令，不许自己执行。

## 踩坑点

1. **`git add -A` 是事故之源。** 第一周 agent 有一次把 `.env.local` 加进了暂存区，幸好拦截规则先跑了一步。后来在 skill 里写死：只 add 明确列出的文件。
2. **agent 会编 message。** 有次 diff 只改了一行配置，message 写出三行"重构"。解法是要求引用 diff 原文，对不上就重来。
3. **分支清理必须二次确认。** `--merged` 在多远端场景下不完全可靠（本地合了但没 push 的情况），清单确认这步不能省。
4. **工作目录漂移。** agent 偶尔在 home 目录执行 git 命令，拿到奇怪的结果。给工具调用固定 `cwd` 参数后消失。

## 可复用建议

- commit 规范放进仓库的 CONTRIBUTING.md，人和 agent 读同一份，规则只维护一处；
- 给 agent 配独立的 git 身份和 SSH key，提交记录一眼可审计；
- 所有写操作走"列计划 → 人工确认 → 执行"三段式，宁可慢一点；
- 每次执行后让 agent 回传命令输出，聊天记录天然就是操作日志。

## 总结

让 AI 管 Git，价值不在"自动"，而在把高频机械操作纳入一套有护栏、可审计的流程。能力上 agent 完全够用，关键是你敢给它多大权限。建议从"只生成 message、只清理分支"这类低风险动作起步，跑稳了再逐步放开 push。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/077052d2a66eef7a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/87033eefd1edd3ee.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/e701b9cb7757cdb3.png)

