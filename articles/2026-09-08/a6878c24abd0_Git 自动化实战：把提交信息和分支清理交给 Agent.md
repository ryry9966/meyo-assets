---
title: Git 自动化实战：把提交信息和分支清理交给 Agent
feedId: 36624
source: 综合讨论
publishedAt: 2026-09-08
---

## 背景

写代码久了会发现，Git 操作里真正费脑子的部分很少：大部分时间耗在写提交信息、切分支、删已合并的旧分支这类重复劳动上。这类工作规则明确、输入输出清晰，恰好是 Agent 最适合接管的场景。在 OpenClaw 生态里，通过 MCP 的 git server，或者给 Agent 一个受限 shell，就能把这件事跑起来。

## 问题

先说清楚要解决什么，避免"为了自动化而自动化"：

1. **提交信息质量不稳定**：`fix`、`update`、`wip` 满天飞，回溯历史时等于没有历史；
2. **分支生命周期没人管**：本地堆着几十个早已合并的分支，`git branch` 一屏放不下；
3. **上下文切换成本高**：改完代码还要自己归纳"这次到底改了什么"，打断心流。

## 做法

**第一步：给 Agent 一个受限的 Git 入口。** 不建议把完整 shell 直接丢给模型，而是把常用操作封装成几个小工具：`stage_and_commit`、`create_branch`、`list_merged_branches`、`delete_branch`。每个工具内部校验参数（分支名格式、仓库路径），并只放行白名单内的子命令：

```text
allow: git status / git diff / git log / git branch / git commit
deny:  git push --force / git reset --hard / git clean
```

**第二步：提交信息生成。** 核心原则是只喂真实 diff，不允许模型凭记忆描述改动。流程固定为：`git diff --staged` → 按 Conventional Commits 规范生成 type/scope/subject → 展示给用户确认 → 确认后执行 `git commit`。确认这步不能省，前期宁可多按一次回车。

**第三步：分支清理。** 让 Agent 定期或手动触发：列出已合并到主干的本地分支 → 排除 `main`/`master`/`dev` 和当前分支 → 输出待删清单 → 人工确认后批量 `git branch -d`。注意用 `-d` 而不是 `-D`，让 Git 自己做安全检查。

## 踩坑点

- **不要给 `push --force` 权限**。自动化阶段，强推永远留在人手里。
- **交互式命令跑不通**。`git rebase -i`、需要输密码的凭据提示都会卡死 Agent，要么换非交互等价命令，要么别自动化。
- **工作目录错位**。Agent 可能在错误的仓库里执行命令，工具内部务必用 `git rev-parse --show-toplevel` 校验路径是否等于预期。
- **index.lock 冲突**。多个 Agent 或人并行操作同一仓库会撞锁，先串行化或加文件锁。
- **模型幻觉改动内容**。如果 diff 为空却生成了提交信息，直接拒绝执行——提交信息必须能和 diff 对上。

## 可复用建议

- 把"生成 → 展示 → 确认 → 执行 → 记录"做成固定管线，所有 Agent 的 Git 动作都走这条链；
- Conventional Commits 是人和 Agent 之间的契约，规范定得越死，自动化越稳；
- 每次 Agent 执行的 git 命令落一条日志，出问题能回放；
- 从本地、低风险操作开始；涉及远端和共享分支的，一律加人工闸门。

## 总结

Git 自动化不值得追求"全自动"，值得追求的是"重复环节归 Agent，判断环节归人"。提交信息生成、分支清理这类活儿，规则清晰、可回滚、风险局部，适合先落地；跑顺之后再往 PR 描述、cherry-pick 同步等场景扩展。先让 Agent 当实习生，别急着让它当发布经理。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/bf64a932c563d766.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/bb2232747179229b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/3e292a4bed060501.png)

