---
title: 让 AI 助手安全地管 Git：提交、分支与 MCP 工具封装
feedId: 32984
source: 综合讨论
publishedAt: 2026-08-13
---

# 让 AI 助手安全地管 Git：提交、分支与 MCP 工具封装

## 背景

在 OpenClaw 这类 Agent 环境里，我们经常希望 AI 不只回答问题，还能直接参与工程流程。Git 提交、分支管理是最常见也最危险的场景之一。如果把 Bash 工具直接开放给 Agent，让它自由执行 `git add -A && git commit && git push`，很容易出现提交含密钥、误推 main、分支命名混乱、force push 覆盖远端等问题。

更务实的做法不是“让 AI 自由跑 Git”，而是把 Git 能力封装成一个受限的工具层，通常通过 MCP Server 或插件形式挂到 OpenClaw/Agent 中。AI 只能调用我们定义好的工具，参数受约束，写操作必须经过确认或 dry-run。

## 问题

直接让 Agent 操作 Git 主要有四类风险：

1. **写操作不可逆**：错误 push 或 force push 会影响远端。
2. **上下文不可控**：大仓库 diff 可能占用大量 token。
3. **凭据与会话问题**：无头环境下 HTTPS 认证容易失败。
4. **并发冲突**：多个 Agent 同时写同一个仓库会触发 `.git/index.lock`。

所以核心思路是：**只读操作放开，写操作收敛，提交前必须有结构化草稿。**

## 做法 / 步骤

### 1. 定义最小 Git 工具集

建议只暴露以下工具，而不是直接给 shell：

| 工具名 | 类型 | 说明 |
|---|---|---|
| `git_status` | 只读 | 查看工作区、暂存区、当前分支 |
| `git_diff_unstaged` | 只读 | 查看未暂存变更 |
| `git_diff_staged` | 只读 | 查看已暂存变更 |
| `git_log` | 只读 | 查看最近提交记录 |
| `git_branch_list` | 只读 | 列出本地/远端分支 |
| `git_branch_create` | 写 | 基于最新 main 创建分支 |
| `git_commit_draft` | 只读 | 根据暂存 diff 生成提交草稿 |
| `git_commit_apply` | 写 | 应用确认后的提交信息 |
| `git_push` | 写 | 推送当前分支到远端 |

所有写工具都要求一个 `dry_run` 参数，默认 `true`。只有当用户显式确认后，才允许真正执行。

### 2. 用 diff 生成提交草稿

不要让 AI 凭空写 commit message。先让工具拿到 `git diff --cached --stat` 和关键文件的 diff，然后按模板生成草稿：

```
<type>(<scope>): <subject>

Why:
- <变更原因>

Changes:
- <变更摘要>
```

`type` 从 `feat/fix/refactor/docs/chore/test` 中选择。AI 需要根据 diff 内容判断类型，如果无法判断，默认用 `chore`，而不是硬猜 `feat`。

草稿只输出，不落地。用户确认后，再由 `git_commit_apply` 执行，工具内部通过文件或参数传递完整 message，避免 shell 拼接问题。

### 3. 分支自动化

分支创建可以按任务标题生成 slug。例如：

- 任务标题：`修复用户登录 token 过期问题`
- 分支名：`fix/token-expiry-login`

规则可以放在工具层：只保留小写字母、数字、连字符，长度控制在 50 字符内。创建分支前先 `git fetch origin main`，再从最新 main 切出，避免基于过期本地分支。

示例流程：

```
git_status → 确认工作区干净
git_fetch → 更新 main
git_branch_create("fix/token-expiry-login", base="origin/main")
```

### 4. OpenClaw 中的接入方式

如果使用 MCP，可以在配置中挂载 Git 工具服务，然后在 Agent 的 system prompt 中规定：

- 只能使用 `git_*` 工具，不直接执行 shell Git 命令。
- 提交草稿必须等用户回复“确认”后才能执行。
- 不允许修改已 push 的历史，不允许 force push。
- 每次写操作前先查看 `git_status`。

这样 Agent 的行为边界就被工具清单和 prompt 双重约束。

## 踩坑点

1. **`git add -A` 提交了密钥或大文件**  
   解决：工具层禁止 `git add -A`，改为由用户或 hook 显式指定文件。或者加 pre-commit hook 扫描 `.env`、私钥等。

2. **diff 过长导致上下文爆炸**  
   解决：先返回 `--stat`，只对变化行数小于阈值的文件取完整 diff。大文件用 `--unified=3` 限制上下文。

3. **`.git/index.lock` 并发冲突**  
   多 Agent 写同一仓库时常见。工具层遇到 lock 错误应返回友好提示，而不是重试或删锁。最好对每个仓库设置单写者队列。

4. **push 时认证失败**  
   无头环境优先用 SSH key，或提前配置 credential helper。不要把 token 写进 prompt 或日志。

5. **自动判断 commit type 错误**  
   只看文件名容易把 `fix` 识别成 `feat`。建议结合 issue 标题或让 AI 给出判断理由，模糊时降级为 `chore`。

## 可复用建议

- **只读优先，写操作确认**：所有写工具默认 `dry_run: true`。
- **提交草稿与执行分离**：先生成 message，再 apply，避免 AI 边写边改。
- **限制工作目录**：每个 Agent 绑定一个仓库根路径，防止在错误目录执行 Git。
- **记录审计日志**：写操作记录 agent id、时间、diff stat、最终 commit hash，便于回溯。
- **利用 pre-commit hook 做最后防线**：AI 生成的提交也要跑 lint、密钥扫描、commit message 格式校验。
- **不要给 AI `rebase` 或 `reset --hard` 工具**：这类操作不属于日常自动化范围，风险远大于收益。

## 总结

Git 自动化不是让 AI 替人执行 Git 命令，而是把 Git 变成一组安全、可审计、可确认的工具。在 OpenClaw/Agent/MCP 场景下，通过工具白名单、dry-run、提交草稿确认和分支 slug 规则，既能让 AI 真正参与代码管理，也能避免把仓库交给一个不受控的 shell。

最实用的起点不是做一个“全自动提交机器人”，而是先实现 `git_status` + `git_diff` + `git_commit_draft` 这三个工具，跑通“分析 → 草稿 → 确认”的最小闭环。稳定之后，再逐步放开分支创建和 push。

---

