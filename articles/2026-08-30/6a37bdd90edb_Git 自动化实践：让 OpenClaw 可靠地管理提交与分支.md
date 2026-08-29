---
title: Git 自动化实践：让 OpenClaw 可靠地管理提交与分支
feedId: 35282
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景

OpenClaw 的 Agent 经常负责改代码、补测试、修配置。真正落地时，`git add/commit/branch` 这些重复动作如果仍靠人手，会打断 Agent 的自动化闭环；如果直接开放 shell 让模型自己执行，又可能出现提交信息混乱、误提交文件、甚至误操作分支。

## 问题

最大问题不是“能不能执行 Git”，而是“没有边界地执行 Git”。常见风险：

- `git add -A` 把 `.env`、lock 文件、临时文件一起提交
- 提交信息随意，难以生成 changelog 或回溯
- 从过期本地 main 切分支，合并时冲突
- 直接调用 `reset --hard`、`push --force` 等危险命令

因此，Git 自动化应该走“受限工具 + 明确策略 + 审计日志”的路线。

## 做法/步骤

### 1. 接入受限 Git 工具

在 OpenClaw 中优先使用 Git MCP，而不是裸 shell。只暴露安全操作：`status`、`diff`、`diff --cached`、`log`、`branch`、`add`、`commit`、`switch -c`、`fetch`、`push --set-upstream`。默认屏蔽 `push --force`、`reset --hard`、`clean -fd`、`checkout -- .`。

示例配置思路：

```json
{
  "mcpServers": {
    "git": {
      "command": "uvx",
      "args": ["mcp-server-git"],
      "env": { "GIT_TERMINAL_PROMPT": "0" }
    }
  }
}
```

### 2. 写清 Git 策略

把下面策略挂到 agent 指令或仓库的 `.openclaw/git-policy.md`：

```text
- 提交前必须展示 git status --short 和 git diff --stat
- 只添加用户指定的路径，不使用 git add -A
- 提交信息遵循 conventional commits，不超过 72 字符
- 分支命名：feat/{issue}-{slug}、fix/{issue}-{slug}、chore/{slug}
- 切分支前执行 git fetch --prune，确认基于 origin/main
- 推送前检查 ahead/behind，落后时先 rebase 或 merge
- 禁止修改 .env、*.pem、*.key、*.lock 的历史
```

### 3. 分支自动化

当用户说“为 #123 创建修复分支”时，agent 执行：

```bash
git fetch --prune
git switch -c fix/123-empty-parser origin/main
```

提交时生成：

```bash
git add src/parser.ts tests/parser.test.ts
git commit -m "fix(parser): handle empty input (#123)"
git push --set-upstream origin fix/123-empty-parser
```

关键不是让 agent“自由发挥”，而是让它按模板执行，并给出可检查的输出。

## 踩坑点

- **交互式 hook 卡住**：pre-commit 等 hook 如果有 prompt，MCP 环境没有 TTY 会挂起。设置 `GIT_TERMINAL_PROMPT=0`，并保证 hook 非交互。
- **agent 选择文件过宽**：不要相信“我会只提交相关文件”。必须在策略里要求先列出待提交文件，人工或规则确认。
- **提交信息漂移**：即使 prompt 要求 conventional commits，长对话后仍可能偏离。最好加 commit-msg hook 做格式校验，模型不合规就拒绝提交。
- **分支基于本地旧 main**：强制 fetch 后再切分支，避免“基于三天前的 main”这类问题。
- **权限过大**：先只读运行一周，记录 agent 所有 git 调用和 diff 摘要，确认无误后再开放写权限。

## 可复用建议

1. 将 Git 策略作为仓库固定文件，让 agent 每次操作前读取，而不是只写在系统提示里。
2. MCP 工具优先于 shell，能拿到结构化结果，减少解析错误。
3. 分支和提交信息模板化：`type(scope): subject (#issue)` 是最低成本的规范。
4. 保留审计日志：记录时间、命令、文件列表、diff stat，便于回溯。
5. 分阶段开放权限：read-only → add/commit → branch/push，稳定一步再走下一步。

## 总结

Git 自动化适合从“规则明确、重复性高”的提交和分支场景切入。把危险命令挡在工具层，把提交规范放在策略层，把异常暴露在审计层，AI 助手才能真正稳定地管理代码提交和分支，而不是制造新的维护负担。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/f33fda3e65a58b3b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/8e7d3232afddef15.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/39581c3a3f2eb6e1.png)

