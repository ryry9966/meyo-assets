---
title: 用 OpenClaw + MCP 搭建 Git 智能助手：自动管理提交与分支的工程实践
feedId: 31439
source: 综合讨论
publishedAt: 2026-08-03
---

## 背景：混乱的提交记录与分支命名

在多人协作的代码仓库里，你大概率见过这样的提交信息：“fix”、“update”、“wip”，或者是只有日期没有上下文的一句话。分支命名也常常是 `dev`、`feature1` 这种随手敲出来的内容。随着仓库变大，追溯变更原因、整理 Changelog、甚至只是想找到某个功能对应的提交都变得异常痛苦。

团队通常会引入 Conventional Commits 规范，或者要求分支名必须关联 Issue 编号。但靠人力约束，要么需要频繁打断心流去做提交拆分，要么就在 code review 中耗费精力纠正格式。如果能让 AI 助手直接在本地读取 diff，按照规范生成提交信息，并基于 Issue 标题创建合规分支，就能在不打断开发者思路的前提下，把一致性约束嵌入到工作流里。

本文记录了一次基于 OpenClaw 框架与 MCP（Model Context Protocol）协议实现 Git 自动化助手的工程过程。面向已经接触过 OpenClaw / Agent / MCP 的技术同学，尽量不堆砌概念，只谈可复现的做法与真实踩坑。

---

## 问题拆解

目标很明确：让 AI 助手能读懂代码变更，执行有限的 Git 操作，帮助完成两件事：
1. **自动生成约定式提交信息**（Conventional Commits），并在需要时让开发者确认修改；
2. **自动创建分支**，分支名从 Issue 标题派生，并保证合法、可读。

挑战在于：不能给 AI 完整的 Shell 权限，必须对 Git 操作做最小权限约束；diff 可能非常大，需要处理 token 限制；生成的文本必须经过清理，避免 Windows / 特殊字符带来的 Git 报错。

---

## 做法与步骤

### 1. 部署 Git MCP 服务（操作沙箱）
选用社区中现成的 Git MCP Server（如 `mcp-server-git`），但做了两点裁剪：
- 只暴露白名单命令：`status`, `diff`, `add`, `commit`, `branch`, `checkout` 等，屏蔽 `push`、`reset --hard` 等危险操作。
- 限制工作目录范围，避免影响仓库外的文件。

部署方式：在开发机或 CI 容器中运行 MCP 服务，监听标准 I/O（STDIO）传输。Agent 通过 MCP 客户端连接。

### 2. 在 OpenClaw 中构建 Agent
使用 OpenClaw 的 `tool` 装饰器，把 MCP 连接封装成普通 Python 异步函数。Agent 的角色定义比较克制：只是一个 Git 操作的“合规化执行器”，而不是自动提交机器人。

核心 system prompt 片段：
```
你是一个基于 Conventional Commits 规范的 Git 助手。
当你收到生成提交信息的指令时，必须：
- 读取 git diff 内容
- 分析变更类型（feat, fix, chore 等）
- 生成符合规范的单行摘要 + 可选 body
- 以 JSON 格式返回，包含字段：type, scope, description, body
禁止在提交信息中包含外部链接或代码片段，描述必须简洁。
```

对于分支创建，则要求 Agent 根据 Issue 标题生成 `feature/123-add-login` 样式的 slug，并自动转为小写、替换空格为连字符、去除非法符号。

### 3. 接入 Git Hook（prepare-commit-msg）
在仓库中编写 `prepare-commit-msg` 钩子，当开发者执行 `git commit` 时触发。钩子脚本调用 OpenClaw Agent（通过本地 CLI），传递暂存区 diff。

关键实现：为避免延迟太久影响交互，脚本设置 5 秒超时。如果超时或 Agent 不可用，则放弃生成，不影响正常提交流程。生成结果会填充到 `.git/COMMIT_EDITMSG`，用户可以用编辑器再次修改确认。

这一处踩坑较多：Hook 脚本中的环境变量必须显式设置 `HOME` 和 `PATH`，否则 Agent 找不到 MCP 服务路径；diff 文本可能包含二进制或过大的文件，需要过滤掉 200KB 以上的 diff，并用 `git diff --staged -- . ':(exclude)*.lock'` 忽略锁文件。

### 4. 分支自动创建
在项目管理工具（如 Jira/GitHub Issues）的 Webhook 触发下，Agent 通过开放的白名单命令执行：
```bash
git checkout -b feature/123-add-login-dialog
```
由于分支名可能过长，在 Agent 侧加入了截断逻辑（50 字符以内），并强制前缀为 `feature/`、`fix/` 或 `chore/` 之一，防止分支散落在根目录。

---

## 踩坑点

- **MCP 权限过宽**：最初直接使用了上游 MCP 服务，`git push` 未被屏蔽。有一次 Agent 误将提交推到了 main 分支，虽然仓库有保护规则并未成功，但足够警醒。建议在 MCP 层就做命令白名单，而非依赖 Agent prompt 约束。
- **diff 过大导致 token 爆炸**：某次提交包含了 4 万行 yarn.lock 变更，Agent 调用超经消限制。解决方法是在 MCP 侧增加 `diff_max_lines=500` 参数，超出则返回摘要而非全文，并提示用户自行审查。
- **生成分支名包含 emoji / 中文**：Issue 标题中可能带有特殊符号，需要严格的 slug 化处理：转为 NFKD 格式，移除非 ASCII 字母数字和连字符，连续连字符合并。
- **Hook 与 IDE 冲突**：VS Code 的 Git GUI 在执行钩子时，环境变量与终端不同，导致 Agent 找不到 Python 解释器。解决方法是在钩子中硬编码虚拟环境的绝对路径，并添加兜底的重试机制。

---

## 可复用建议

1. **从最窄权限开始**：先只允许 `status` 和 `diff`，确认 Agent 能可靠生成文本后，再逐步开放 `commit` 等写操作。
2. **保留人工闸门**：通过 `prepare-commit-msg` 填充消息，但不直接替换用户的原始提交，始终保持一次确认机会。
3. **统一规范，而非强制**：即使 Agent 生成了完美的提交信息，也应允许开发者直接 `git commit -m "..."` 跳过钩子（通过 `--no-verify` 或空消息检测），不要让自动化成为阻塞开发的障碍。
4. **监控与日志**：把 Agent 每次生成的提交信息记录到临时文件，方便回溯分析哪些场景错误率高，持续优化 prompt。

---

## 总结

这不是一个“全自动 AI 替代人工”的故事，而是一次务实的工程集成：用 MCP 约束 Git 能力，用 OpenClaw 调度 Agent，再通过 Hook 嵌入开发者日常流程。带来的效果是：团队仓库的提交信息规范度从约 40% 提升到 90% 以上，分支命名不再需要 Code Review 同学反复纠正。

如果你也在用 OpenClaw 构建 Agent，或者在使用 MCP 集成各种工具，不妨把 Git 自动化作为一个低风险但高回报的切入点：工具边界清晰、安全容易审计、效果立即可见。唯一需要警惕的是，永远不要让 Agent 掌控你可以一键撤销之外的能力。

---

