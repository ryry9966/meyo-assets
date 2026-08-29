---
title: Git 自动化实践：让 AI 助手管理提交和分支的约束设计
feedId: 35181
source: 综合讨论
publishedAt: 2026-08-29
---

## 背景

在 OpenClaw、Agent、MCP 的自动化环境里，Git 常被当成“顺手接一个工具”。但如果直接把 shell 或通用 Git MCP 暴露给模型，风险大于收益。`reset --hard`、`push --force`、`branch -D` 都不可逆；一次误提交可能把密钥、大文件或半成品代码带进历史。可落地的 Git 自动化，不是让 AI 自由执行命令，而是把操作拆成可审计、可确认、有边界的工具。

## 问题

直接开放 Git 给 Agent，常见问题包括：读到 `.env`、密钥文件并生成泄密提交；在错误目录或分支执行；commit message 空泛；分支命名随意；一次提交混入大量无关文件；误用 push 或删除未合并分支。

## 做法/步骤

我采用“只读优先、写操作显式确认”的方式封装 Git。可做成 MCP 工具或插件命令：

1. 默认只开放只读命令：`git status --porcelain`、`git diff --stat`、`git diff -- <file>`、`git log --oneline -10`、`git branch --show-current`。
2. 写操作仅保留 `add`、`commit`、`checkout -b`、`branch -d`，且通过函数参数调用，不直接给 shell。禁止 `reset --hard`、`push`、`clean`、`push --force`。
3. 提交信息模板化：`type(scope): summary` + body 说明 why。要求先读 diff 再生成 message，不允许在看不到 diff 的情况下提交。
4. 示例流程：Agent 调 status 看到 `M src/auth.py` 和 `M tests/test_auth.py` → 读 diff → 输出 plan：branch 为 `fix/auth-token-refresh`，message 为 `fix(auth): refresh token on 401`，risk 为 none → 人工确认 → 依次执行 `checkout -b`、`add` 指定文件、`commit` → 最后用 `log -1` 校验提交内容，确认没有夹带其他文件。

## 踩坑点

- **diff 过大**：几万行 diff 会占满模型上下文，需限制 diff 行数或按文件拆分提交。
- **pre-commit hook 反噬**：格式化工具可能自动修改文件，Agent 若再次 `add`，会把计划外内容混入提交。需重新展示 diff 并确认。
- **仓库路径与凭据**：不要给 HOME 目录级 Git 权限，固定工作目录；凭据不要放在 Agent 可读环境变量中。
- **分支清理**：用 `branch -d` 而非 `-D`，确认已合并后再删。
- **shell 注入**：分支名、commit message 不要拼接进 shell，用参数数组调用 Git，避免命令注入。

## 可复用建议

- MCP 工具拆分 read/write 两组，write 默认关闭，plan 确认后启用。
- pre-commit 作最后防线，检测密钥、大文件、格式。
- system prompt 明确禁止提交 `.env`、`node_modules`、密钥、大文件，禁止 push、force、reset --hard，单次提交不超过 5 个文件。
- 记录操作日志：commit hash、分支、文件列表写入 JSON，方便回溯。
- 小步提交：一个逻辑变更一个 commit，避免把无关修改混入特性提交。

## 总结

把 Git 交给 AI 管理的核心不是“能否执行命令”，而是“每一步都可确认、可回滚、可审计”。只读优先、写操作 plan 确认、模板化提交、hook 兜底，才能让自动化真正进入日常开发流程，而不是制造新的故障点。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/1d6b4e9fc8dc1cbe.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/dc1543b0692444b7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/7389f6e4cb528cc5.png)

