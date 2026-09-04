---
title: Git 自动化：把提交与分支的机械活交给 AI 助手
feedId: 36027
source: 综合讨论
publishedAt: 2026-09-04
---

## 背景

日常开发里，Git 操作占了大量琐碎时间：拆暂存区、写规范的 commit message、给功能开分支、清理已合并的旧分支。OpenClaw 这类带工具调用能力的 Agent 接入 MCP git 工具（或受控 shell）之后，这些活完全可以交给助手做，人只负责 review 和拍板。

## 问题

直接丢一句"帮我提交"通常会翻车，常见三类：

1. **提交信息与改动不符**——Agent 只看了部分 diff，message 全是泛泛而谈；
2. **一把梭 `git add .`**——把 `.env`、日志、构建产物一起带进提交；
3. **危险操作不受控**——force push、在共享分支上 rebase、在 detached HEAD 上乱提交。

## 做法

一套我们验证过的流程：

**1. 收敛工具面。** 优先用 MCP 的 git server，只开放 `status` / `diff` / `add` / `commit` / `branch` / `push`；走 shell 就上命令白名单，deny 掉 `push --force`、`reset --hard`、`clean -fd`。

**2. 把约定写进规则文件。** 仓库根放一份 AGENTS.md，明确：Conventional Commits 格式；分支命名 `feature/xxx`、`fix/xxx`；提交前必须读全量 diff；禁止提交密钥和产物；只在自建分支工作。

**3. 固化工作流。** 让 Agent 每次按同一顺序执行：

- `git status` + 全量 `git diff` 检查；
- 判断是否需要拆成多个提交；
- 逐个 stage 相关文件（明确列文件名，不用 `add .`）；
- 生成 message，附一行改动摘要给人确认；
- 在 feature 分支提交并 push，最后输出 PR 链接。

**4. 用工程手段兜底。** pre-commit 跑 lint 和密钥扫描，分支保护规则限制直推 main。Agent 想改写历史？分支保护直接拒绝。这层护栏不靠 prompt 自觉。

## 踩坑点

- **部分 diff 写 message**：Agent 为省 token 只看 `--stat`，输出全是空话。规则里写死"必须读全量 diff"后，message 质量明显提升。
- **hook 失败死循环**：pre-commit 拒绝后 Agent 盲目重试。要求它先读 hook 输出、修复再重试，超过 2 次就停下问人。
- **凭证暴露**：Agent 进程能读到 git credential。挂最小权限的 scoped token，别把有组织写权限的 SSH key 给它。
- **基线漂移**：长命分支不同步 main，Agent 合并时自作主张解冲突。约定：冲突超过 3 个文件就停下汇报。

## 可复用建议

- 规则写在仓库文件里，跟着代码走、能被 review，比塞在全局 system prompt 里可维护得多。
- 让 Agent 先输出"操作计划"再执行，push 默认要人确认，跑顺了再逐步放开。
- 提交粒度交给它拆：一次一个逻辑变更；你只审 message 与 diff 的一致性。
- 把"清理已合并分支"做成可复用的 agent 任务定期跑，注意排除未合并的 WIP 分支。

## 总结

Git 自动化的收益不在"全自动提交"，而在把机械劳动外包、把决策权留在人手里。真正的护栏是工具白名单、hook 和分支保护，不是提示词里的一句"请小心"。Agent 负责读 diff、拆提交、写 message，人负责确认与合并——这个分工跑顺之后，提交历史会干净很多，回溯成本也低得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/186b7db8f016f1e1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/c07669254ceb1e36.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/16d6ad26b1bf8b7e.png)

