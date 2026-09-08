---
title: Git 自动化实战：把提交与分支交给 OpenClaw Agent
feedId: 36645
source: 综合讨论
publishedAt: 2026-09-08
---

## 背景

用 OpenClaw 这类 Agent 写代码之后，提交频率明显上来了。AI 生成的改动常常一次横跨多个文件，如果还靠手工 `git add` + 手写 commit message，一天下来 Git 记录会很难看，分支也越积越多。Git 本身命令面很完善，缺的是“谁来决策、谁来执行”这一层——这正好是 Agent + MCP 的用武之地。

## 问题

实际痛点集中在三处：

1. **提交信息质量差**。赶工时的 "fix"、"update" 一多，回溯历史全靠考古。
2. **分支无人清理**。合并完的分支留在本地，几周后 `git branch` 一屏都放不下。
3. **上下文切换成本高**。改着 A 分支想起 B 分支有个 hotfix，来回 checkout 容易把未提交的改动带乱。

## 做法

我的方案是给 OpenClaw 挂上 Git MCP server，再加两个自定义 skill：

**第一步：接入 Git 工具链。** 在 OpenClaw 配置里启用 Git MCP server，暴露 `status`、`diff`、`commit`、`branch`、`log` 这类只读加低风险写操作。push、reset、clean 默认不开。

**第二步：写一个 /commit skill。** 流程固定为：`git status` 确认暂存区 → `git diff --staged --stat` 看规模 → 必要时抽查关键文件 diff → 生成符合 Conventional Commits 的 message → 提交前把 message 和文件清单贴给用户确认。实测下来，九成提交信息的质量比手写稳定。

**第三步：分支周清。** 建一个定时任务，每周让 Agent 跑 `git branch --merged`，列出可删分支，逐条询问后执行 `git branch -d`（注意只用小写 `-d`，不碰 `-D`）。

## 踩坑点

- **危险命令必须白名单化**。我有次让 Agent“清理工作区”，它直接给出 `git clean -fd` 的方案。后来把允许的子命令写死在 system prompt 里，远程和破坏性操作一律二次确认。
- **大 diff 撑爆上下文**。别让 Agent 一上来读全量 diff，先 `--stat` 再定向看文件，context 能省一半以上。
- **rebase 冲突别让 Agent 猜**。约定遇冲突立即 `git rebase --abort` 并报告，而不是自行取舍代码——它替你选的版本你不敢直接信。
- **staged 与 unstaged 混淆**。Agent 有时会把没暂存的改动一起提交，skill 里要显式校验 `status` 输出再动手。

## 可复用建议

1. 保护分支列表（main、release/*）写进配置，任何操作前先校验。
2. Agent 执行的所有 Git 命令落日志，出问题能回放。
3. 破坏性操作统一走“先 dry-run、再确认”两段式。
4. 把团队 commit 规范做成 few-shot 示例塞进 skill，比讲道理有效。

## 总结

Git 自动化的价值不在省那几秒敲命令，而在于把“提交规范、分支卫生”这些容易烂尾的事变成 Agent 的固定动作。原则只有一条：**让 Agent 做决策判断和重复劳动，把不可逆操作留在人手里**。这套配置迁移成本很低，一个 MCP server 加两个 skill 就能跑起来，建议从 `/commit` 开始试。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/f4e6e131495b3fdf.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/a212b2429bba4362.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/47078742569b83d6.png)

