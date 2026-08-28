---
title: Git 自动化：让 AI 助手管理代码提交和分支的工程化实践
feedId: 35135
source: 综合讨论
publishedAt: 2026-08-29
---

## 背景

在 OpenClaw 的 Agent 工作流里，越来越多任务是“修改代码 → 切分支 → 提交 → 提 PR”。当 Agent 生成完代码后，手动执行 git add/commit 会打断自动化。于是很多实践者会让 Agent 直接调用 shell 执行 git 命令。

问题是，直接放开 shell 权限后，Git 操作往往成为风险最高的环节之一。

## 问题

常见故障包括：在 main/master 上直接提交；临时文件、日志、.env 被一并提交；提交信息不稳定，时而过长，时而只有 "update"；分支名随意；Agent 在冲突时反复重试 rebase，甚至误执行 force push。

这些问题的根源不是 Agent 能力不足，而是 Git 工具没有被约束成“可审计、可回滚、有边界”的操作集。

## 做法

### 1. 把 Git 封装为受限工具/MCP，而不是开放 shell

在 OpenClaw 里，可以把 Git 操作挂成 MCP server 或插件工具，只暴露安全子集。示例配置：

```json
{
  "workdir": "/repo",
  "allowed": ["status", "diff", "add", "commit", "branch", "checkout", "log"],
  "denied": ["push --force", "reset --hard", "rebase", "branch -D"],
  "commit": {
    "types": ["feat", "fix", "docs", "refactor", "test", "chore"],
    "max_subject_len": 72
  }
}
```

工具内部负责校验参数、检查工作区状态、限制工作目录范围。

### 2. 制定提交信息规范

不建议让 Agent 自己凭空写 commit message。工具可以先把 `git status --short`、文件列表、变更摘要和分支名传给 Agent，再要求它按 Conventional Commits 生成。工具校验 type、scope、subject 长度，不符合就拒绝提交并返回结构化错误。

这样多次提交后，提交历史会稳定很多，也便于后续生成 changelog。

### 3. 分支策略

默认从最新 main 拉取 `feature/ai/<task-id>` 分支。工具只允许在某类临时分支上提交，禁止直接在 main/production 分支提交。合并时只生成 PR 草稿，不自动 push main。删除分支只允许删除 `feature/ai/` 前缀且无未合并提交的分支。

### 4. 门禁与审计

在 add/commit 前做 preflight：检查敏感文件、冲突标记 `<<<<<<<`、大文件、是否在保护分支。每次 Git 操作写一行 JSONL 审计日志，包含 session_id、工具名、参数、结果摘要和 commit hash。

## 踩坑点

1. **不要传全量 diff 给 Agent**。大改动会撑爆上下文，导致提交信息失真。工具应只发送 diff stat、文件列表和截断到 200 行左右的差异摘要。

2. **敏感文件检测必须在 add 前做**。`.env`、`*.pem`、`id_rsa`、`credentials` 等文件一旦提交，后面再删历史也会留下。工具应直接阻止 add/commit。

3. **提交钩子可能拖垮 Agent 调用**。pre-commit 里跑 lint、test 时，工具调用会卡住直到超时。轻量检查走同步，重检查交给 CI。

4. **冲突不要自动 rebase**。Agent 遇到冲突后如果允许它反复执行 rebase/merge，容易把分支状态搞乱。遇到冲突应立即停止，返回冲突文件列表，转人工。

5. **多个 Agent 或人并发操作同一分支**。工作区脏时不执行 pull/checkout；需要切换分支时先检查是否有未提交改动。

## 可复用建议

- **默认 dry-run**：commit、push、merge 等操作先预览，再执行。
- **不可逆操作人工确认**：force push、hard reset、删除分支、rebase 不应自动执行。
- **返回结构化错误**：不要只返回 "failed"，要让 Agent 知道是“工作区脏”“提交信息不合规”还是“敏感文件被拦截”。
- **保留会话级审计**：每条操作日志都应包含 Agent session_id，方便定位是哪次自动化改动。
- **把规则写进 system prompt**：明确告诉 Agent 哪些分支能提交、提交信息格式、遇到冲突停止，比等到工具报错再纠正更稳定。

## 总结

Git 自动化不是让 Agent 替人敲命令，而是把 Git 能力变成一组有边界、能审计、可回滚的工具。OpenClaw 用户通过 MCP/插件方式接入后，重点不是“能不能自动提交”，而是“提得对不对、能否定位、能否回滚”。

把权限收窄，提交信息规范，分支策略固定，再加上审计日志和关键操作确认，Git 自动化才能真正进入工程化流程。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/177e383e8f370fe6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/2546be1426712bde.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/43da5a0474afb5a4.png)

