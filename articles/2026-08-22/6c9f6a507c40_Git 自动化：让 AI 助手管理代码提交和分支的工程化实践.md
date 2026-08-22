---
title: Git 自动化：让 AI 助手管理代码提交和分支的工程化实践
feedId: 34192
source: 综合讨论
publishedAt: 2026-08-22
---

## 背景

日常开发里，大量 Git 操作是机械且重复的：看 diff 写 commit message、切分支、合并后清理分支、整理 changelog。这些工作规则明确、不依赖复杂业务判断，很适合交给 AI 助手。尤其在 OpenClaw + Agent + MCP 的工作流里，与其让模型“假装”执行命令，不如把 Git 能力封装成受控工具，真正跑在本地仓库上。

但问题也很直接：Git 是危险操作集。`reset --hard`、`push --force`、误 checkout、把 `.env` 提交上去，任何一次误判都可能造成真实损失。所以关键不是“让 AI 能跑 Git”，而是“用工程化方式限制 AI 怎么跑 Git”。

## 做法：把 Git 封装成受限工具

在 OpenClaw 里，我建议不要直接开放 shell 工具，而是通过 MCP server 或插件暴露一组白名单 Git 工具。每个工具只做一件事，参数严格校验。

推荐暴露的最小工具集：

- `git_status`：查看工作区状态
- `git_diff`：查看未暂存/已暂存 diff，支持文件级过滤和行数上限
- `git_log`：查看最近提交
- `git_branch_list`：列出本地/远程分支及合并状态
- `git_add`：仅添加指定文件，不允许 `-A` 或 `.`
- `git_commit`：用传入 message 提交，要求 subject 不超过 72 字符
- `git_create_branch`：从当前分支创建并切换到新分支
- `git_checkout`：切换分支，要求工作区干净或显式 stash
- `git_branch_clean_dry_run`：列出已合并到 main 的分支，不实际删除

注意，上面刻意没有 `push`、`force push`、`rebase`、`reset --hard`。推送可以后续加，但最好先做到“AI 只负责本地操作，推送由人或 CI 完成”。

实现层面，MCP server 的参数定义建议用 JSON Schema，并且所有命令调用使用参数数组形式，而不是字符串拼接。例如：

```json
{
  "name": "git_commit",
  "arguments": {
    "message": "fix: correct login token refresh",
    "files": ["src/auth.ts"]
  }
}
```

不要做成 `exec("git commit -m '" + message + "'")`，否则 commit message 里带单引号或反引号就会变成 shell 注入。

## 提交与分支工作流

一个比较稳的自动提交流程是：

1. AI 先调用 `git_status` 和 `git_diff`，了解改动范围。
2. 根据 diff 生成 Conventional Commits 风格的 message。
3. 输出执行计划：将要 add 哪些文件、commit message 是什么、当前分支是否正确。
4. 用户确认后，AI 调用 `git_add` 添加指定文件，再调用 `git_commit`。
5. 提交完成后，回读 `git_log` 做校验。

分支管理也类似。比如清理已合并分支：

1. 调用 `git_branch_list`，找出已合并到 main 且非保护分支的项。
2. 输出待删除列表。
3. 用户确认后才执行删除。

创建分支则可以结合 issue 标题或需求描述，让 AI 生成 `feat/xxx`、`fix/xxx` 形式的分支名，避免手写分支名不一致。

## 踩坑点

**Shell 注入**  
前面说过，所有命令参数必须走数组。不要因为“就传个字符串”而图省事。

**工作区污染导致切分支丢改动**  
AI 在 `git_checkout` 前必须检查 `git_status`。如果工作区不干净，要么先 `stash`，要么停止操作并提示用户。这是最容易翻车的点。

**大 diff 直接把上下文打爆**  
一次 `git diff` 可能几万行。MCP 工具层要截断，比如每个文件最多返回 300 行，总行数上限 2000。模型不需要看完整 diff，能判断改动意图就够了。

**误提交敏感文件**  
即使有 `.gitignore`，也可能有漏网之鱼。可以在 pre-commit hook 里做 secret scan，AI 提交同样会触发 hook。这样不依赖模型自觉。

**AI 产生碎片提交**  
如果每次小修改都让 AI commit，历史会非常碎。建议设置触发条件：只处理用户主动发起的 `/commit` 或 `/auto-commit` 指令，不要让 AI 在后台自己决定什么时候提交。

**推送与认证**  
尽量不要让 AI 持有 push 权限。本地提交坏了还能 `git reflog` 救回来，推到远端就麻烦得多。分支保护规则也要在 GitLab/GitHub 侧配置，不能只靠 prompt 约束。

## 可复用建议

1. **所有变更类命令先 dry-run**：`git add --dry-run`、`git branch_clean_dry_run` 都是低成本保险。
2. **固定 system prompt 模板**：要求 AI 在执行任何变更前必须输出“文件列表 + message + 当前分支 + 风险点”，否则拒绝执行。
3. **把常见流程封装成 OpenClaw skill 或 slash command**：例如 `/commit`、`/branch-clean`、`/release-note`，用户一键触发，避免每次重新描述规则。
4. **记录执行日志**：把 AI 调用的 Git 命令和参数写进日志文件，出问题能回溯。
5. **配合 pre-commit hook**：格式化、lint、secret scan 都在提交前自动跑，AI 提交和人工提交享受同样的质量门槛。

## 总结

Git 自动化不是让 AI 代替开发者做决策，而是把 diff 阅读、message 生成、分支清理这类机械动作封装成受控工具。工程化的核心只有三点：白名单工具集、dry-run 加确认、变更命令参数化。做到这些，AI 就能安全地帮你把 Git 的日常琐事接过去。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/6e79f6537157c191.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/c8824722d3cc0b1b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/fb1ca43921c957b6.png)

