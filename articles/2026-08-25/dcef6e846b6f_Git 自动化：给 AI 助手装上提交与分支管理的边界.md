---
title: Git 自动化：给 AI 助手装上提交与分支管理的边界
feedId: 34669
source: 综合讨论
publishedAt: 2026-08-25
---

## 背景
在 OpenClaw-CN 的自动化工作流里，AI 助手已经会实际改代码、写脚本、更新文档。尴尬的是：模型改完文件后，流程常卡在 Git 操作上。你得手动检查 diff、写 commit message、切分支，小改动累积起来很耗神。于是自然会想：能不能让 agent 自己完成提交和分支管理？

## 问题：不是不能做，而是容易做过头
直接给 agent 一个 shell 执行任意 Git 命令，风险很高：误提交 .env、密钥；生成“更新代码”“fix bug”这类无信息量提交；把临时文件、日志、二进制一起带入；自动 push 到 main；分支命名随意。我们要的不是“让 AI 执行 Git”，而是“给 AI 一个受控的 Git 工作流”。

## 做法：把 Git 能力封装成受限工具
建议不要直接暴露 shell，而是通过 MCP 接入 Git server，或在 OpenClaw 中封装几个工具。最小集合：

- git_status：查看当前分支和工作区状态
- git_diff：查看未暂存/已暂存变更
- git_branch_create：基于当前分支创建特性分支
- git_commit：在指定路径上创建提交
- git_log：查看最近提交历史

关键设计：push、merge、rebase 不放进工具列表，或必须人工确认。

再给 agent 固定提交规范：

```text
你是代码维护 agent。Git 操作规则：
1. 只提交与当前任务相关的文件，不顺手提交格式化、调试代码或临时文件。
2. 使用 conventional commits：feat/fix/refactor/docs/chore 等类型。
3. 不在 commit message 中写“更新代码”“fix bug”。
4. 不执行 push、merge、rebase，不修改 .git/config。
5. 遇到二进制文件或单文件超过 1MB，停止并报告。
6. 提交前必须基于 git diff 给出变更摘要。
```

典型流程：
1. agent 完成文件修改；
2. 调用 git_status 和 git_diff 查看变更；
3. 根据 diff 判断是否拆分为多个提交；
4. 创建分支，例如 `feat/auto-docs-update`；
5. 执行 git_commit，message 符合规范；
6. 停止，等待人类 push 或开 PR。

## 踩坑点
**中间产物被一起提交**：模型可能创建临时脚本、日志。自动 git add . 会把它们全带入。需要显式指定路径或配置 allowlist/denylist，提交前让 agent 先报告文件列表。

**一次提交混杂多个逻辑**：如果任务改了多个模块，模型容易全塞一个 commit。可在 prompt 中要求“变更涉及两个以上独立主题时，先分组提交”。

**分支切换误伤工作区**：有未提交改动时切分支容易冲突。建议任务开始前工作区必须干净，或让 agent 在独立 worktree 中操作。

**自动 push 风险**：即使只在 feature 分支提交，自动 push 也可能覆盖远程历史。一般只允许本地提交，推送由人确认；若要自动化，只允许 push 到 `feature/*`，禁止直接 push 到 main。

**本地与远端不同步**：只 commit 不 push 会让后续 agent 或协作者看不到最新提交。明确任务是“本地提交 + 报告”，推送由人完成。

## 可复用建议
- 封装成 OpenClaw 插件或脚本，提供 dry-run 模式，先输出将执行的 Git 命令和提交信息。
- 保留审计日志：时间、分支、提交哈希、变更文件列表和 diff 摘要。
- 复用 pre-commit、commitlint、secret scanning，别把防线全交给模型。
- 限制可写路径和分支，例如只允许操作 `docs/`、`scripts/` 或 `feature/auto-*`。
- 从低风险仓库开始：文档、配置、脚本，跑顺后再考虑业务代码。

## 总结
AI 管理 Git 提交和分支，价值在于减少事务性操作：整理 diff、写规范提交、创建分支。合并、推送、冲突解决这些决策性操作仍应留在人手里。自动化不是无人化，而是把重复操作交给工具，把关键节点留在人手里。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/c5f04a48116aede9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/3fa6c7b24e6e9e6a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/5eadfddf55b76725.png)

