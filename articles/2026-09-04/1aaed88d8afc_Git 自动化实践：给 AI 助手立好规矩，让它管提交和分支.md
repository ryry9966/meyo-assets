---
title: Git 自动化实践：给 AI 助手立好规矩，让它管提交和分支
feedId: 36019
source: 综合讨论
publishedAt: 2026-09-04
---

## 背景

写 commit message、切分支、清理合并完的旧分支，这类工作高频、低创造性、规则明确，是 Agent 自动化的理想切入点。OpenClaw 侧挂上 MCP 或 shell 工具后，理论上让助手直接执行 `git` 命令就行。但实践下来，直接给一个裸 shell 风险不小：误推、force push、把 `.env` 一起提交，都是真实发生过的坑。这篇文章记录一套约束分层的做法。

## 问题

三个核心痛点：

1. commit message 质量不稳定，风格随模型心情波动；
2. 分支越积越多，命名混乱，谁都不敢删；
3. Agent 执行 git 命令不可控，提示词约束容易被"忘记"。

## 做法

核心思路是三层结构：**约定放提示词，权限放脚本，安全放 hooks**。

### 1. 用包装脚本收权

不暴露裸 shell，只通过一个白名单脚本开放高级操作（可作为 MCP 工具或 shell tool 挂载）：

```bash
#!/usr/bin/env bash
# git-agent.sh — 只允许白名单操作
case "$1" in
  status)     git status --short ;;
  diff)       git diff --staged ;;
  commit)     git commit -m "$2" ;;
  branch-new) git switch -c "$2" ;;
  push)       git push -u origin HEAD ;;  # 不透传任何用户参数
  *) echo "denied"; exit 1 ;;
esac
```

关键点：`push` 不接受参数，从根上消灭 force push；rebase、reset 这类破坏性命令根本不在名单里。

### 2. 提示词写死约定

- 提交信息遵循 Conventional Commits（feat/fix/chore/docs/refactor）；
- 分支一律 `agent/` 前缀，永不直接提交 main；
- 提交前必须先执行 status + diff，只 stage 自己改过的文件；
- 每次操作后用 `git log -1 --format=%H` 回读 hash 作为执行凭证。

### 3. hooks 兜底

pre-commit 挂 gitleaks 扫敏感信息，commit-msg 校验格式，远端开启分支保护。提示词会被模型忽略，hook 不会。

### 4. 分支清理做成周期任务

让助手定期跑 `git branch --merged main` 加最后提交时间，输出候选删除列表。**默认只报告不删除**，人工确认后走单独的清理命令，且带 `--dry-run`。

## 踩坑点

1. **谎报成功**：助手说"已提交"，实际命令没执行或失败。解法：脚本失败必须非零退出，约定 agent 必须回报 hash，拿不出 hash 视为失败。
2. **stage 夹带私货**：把本地配置、密钥一起提交了。除了 gitleaks，脚本层再硬编码一份 ignore 校验。
3. **交互式命令卡死**：`rebase -i`、`git add -p` 需要 TTY 交互，agent 要么卡住要么瞎编输入。直接在白名单禁掉。
4. **提交粒度失控**：一个小任务拆出十几个 "update" commit。约定里明确"一个逻辑变更一个 commit"。
5. **遇到冲突想强推**：规则改成冲突即 abort 并报告，人来裁决。

## 可复用建议

- 让每个动作产出**机器可验证的证据**（hash、分支名、退出码），而不是自然语言汇报；
- 危险操作一律 dry-run 默认，只建分支、不销毁历史，保留回退路径；
- 所有 agent 的 git 操作追加写入日志文件，方便事后审计；
- 这套"提示词定约定、脚本收权限、hooks 做防线、日志留审计"的四件套，可以直接迁移到其他自动化场景。

## 总结

Git 杂务是 Agent 自动化性价比最高的起步场景。真正的工程价值不在"AI 会写 commit message"，而在约束分层：用提示词管理约定、用脚本管理权限、用 hooks 做最后防线、用日志支撑审计。把这套骨架搭好，再逐步放开更多自动化操作，才走得稳。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/cb95de50294237cf.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/0e8528f7eb1ef7f0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/c3f21d73d936377e.png)

