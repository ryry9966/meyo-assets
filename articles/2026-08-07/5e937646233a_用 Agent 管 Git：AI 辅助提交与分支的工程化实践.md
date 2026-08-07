---
title: 用 Agent 管 Git：AI 辅助提交与分支的工程化实践
feedId: 31966
source: 综合讨论
publishedAt: 2026-08-07
---

## 背景：当代码变更速度超越人工管理能力

在 AI 辅助编程越来越普及的日常里，我们经常会进入一种状态：Agent 帮忙产出代码片段的频率，已经明显快过你手工写 commit message 或切分支的速度。一段时间下来，仓库里难免出现大量 "update", "fix typo", "wip" 这类低信息量的提交，分支命名也是随便起一句就推上去。这在初期似乎无害，但当需要回溯变更、生成 changelog、或多人协作时，代价就开始暴露。

OpenClaw 等 Agent 平台通过 MCP (Model Context Protocol) 可以接入外部工具，其中就包括 Git MCP server。这意味着我们可以让 Agent 不仅产出代码，也能按工程规范帮你完成提交和分支管理，减少机械性操作，同时约束生成结果符合 team 约定。

但这并不等于“AI 替你做决策”。正确的定位是：让 Agent 做信息整理和格式生成的脏活，人类保持审查和最终执行权。下面所述的做法，来自我们在多个内部项目中使用 OpenClaw + Git MCP 的实践，把可复现的步骤、踩过的坑、仍然有效的边界整理出来。

## 要解决的两个高频问题

1. **提交信息随意**：没有遵循 conventional commits，无法自动生成 changelog，也难以通过 git log 快速理解变更意图。
2. **分支管理散乱**：临时修复没有从正确的基准分支切出，命名不规范（比如 fix/bug123、patch-1），合并后难以追踪。

我们希望 Agent 做到：根据实际的 `git diff` 内容，生成符合规范的提交信息；根据任务描述自动创建命名的 feature/fix 分支；在操作前后提供足够的上下文让人类确认。

## 做法与步骤

### 1. 搭建 Git MCP Server
- 使用 `@anthropic/mcp-server-git` 或社区实现的 Git MCP Server（如 `mcp-git-server`），将其作为 OpenClaw 的 MCP 工具提供者。
- 在 OpenClaw 的配置文件中注册工具，确保 Agent 可获得 `git_status`, `git_diff_staged`, `git_add`, `git_commit`, `git_branch`, `git_checkout` 等能力。
- 运行该 MCP server 时会暴露对应的工具接口，OpenClaw 会话能直接调用。

### 2. 配置 Agent 的 system prompt 与权限边界
在 Agent 定义里加入明确的 git 处理规则，例如：

```text
当用户请求“提交更改”时：
1. 先执行 git diff --staged 查看暂存区变更。
2. 根据变更内容，生成符合 conventional commits 的提交信息，格式：type(scope): description。
   type 限定为 feat, fix, refactor, test, docs, chore。
3. 将生成的信息展示给用户，得到确认后再执行 git commit -m "..."。
4. 如果用户未暂存任何文件，先提示暂存或询问是否自动 git add -A。
```

对于分支：

```text
- 用户描述一个任务时，自动生成分支名，格式：type/issue-id-short-desc。
  type 选取 feat 或 fix，issue-id 从上下文中提取（如 #42），若无则省略。
- 切分支前检查当前工作区是否干净，否则提示先处理未提交变更。
- 永远不要强制推送到 main 或 master 分支；仅允许在 feature/fix 分支上推送。
```

这些约束通过 system prompt 固化下来，避免 Agent 越权操作。

### 3. 实际交互示例（与 OpenClaw 对话）

用户：*“帮我提交暂存区的变更，关联 issue #173。”*

Agent 执行 `git diff --staged`，分析输出后生成：

> 提议的提交信息：
> `fix(auth): handle expired token gracefully for oidc flow (#173)`
> 是否确认提交？

用户确认后，Agent 执行 `git commit -m "..."`，并返回结果。

类似地，用户说：“从 main 切一个修复分支，修复登录页样式错乱”，Agent 会生成分支名 `fix/173-login-layout-glitch`，切过去后给出提示。

### 4. 可选增强：结合 pre-commit hook 做二次检查
为了防止 Agent 偶尔生成不规范的提交信息（比如 description 过长或缺失 scope），我们保留了一个轻量的 commit-msg hook，用正则检查 conventional commits 格式。如果不符合，commit 会被拒绝，Agent 会收到错误，并可以在对话中重新修正。

## 踩坑记录

- **Git 环境缺失**：MCP server 运行在与 OpenClaw 相同的环境中，如果容器内缺少 git 或未配置 user.name/user.email，提交会失败。需要预先在启动脚本中设置好全局 git config。
- **SSH 密钥不可用**：如果仓库使用 SSH 方式访问，Server 需要能访问对应私钥。我们将其挂载到容器并配置 `GIT_SSH_COMMAND` 来避免交互式 prompt。
- **Agent 可能会“过度提交”**：早期未限制时，Agent 偶尔会在生成 commit 后自动 push，这非常危险。我们在 system prompt 中明确：除非用户明确说 "push"，否则不执行推送动作。更好的做法是通过工具权限控制，只暴露 `git_commit`，不暴露 `git_push`，需要时由人手动推送。
- **分支冲突处理弱**：当分支已存在或当前分支不是预期的基准时，Agent 并不能很好地解决冲突，往往只是报错。此时需要人工介入。我们为 Agent 增加了一段指引：遇到冲突时，列出具体文件冲突，建议用户手动解决后重新操作。
- **Commit message 生成质量不稳定**：在没有 scope 上下文时，Agent 容易生成过长的描述。通过 prompt 限制“用一句话归纳核心变更，不超过 72 字符的 subject”并加入示例，显著提升了规范性。

## 可复用建议

- 将上述 system prompt 片段抽象成一个 `git_rules.md` 模板，不同项目的 Agent 直接引用，减少重复配置。
- 为团队提供一个“安全模式”开关：通过环境变量 `AI_GIT_AUTOCOMMIT=false` 来控制 Agent 是否可直接执行写操作，这样在初期评审阶段可以只读建议而不实际提交。
- 把常用的分支命名惯例、commit 类型说明写入项目根目录的 `.agent-git-context` 文件，让 Agent 每次操作前读取，适应不同项目规范。
- 如果你的 OpenClaw 工作区可以持久化记忆，让 Agent 记住上次使用的 issue id，以便生成连续操作时分摊引用。

## 总结

用 Agent 管理 Git 并没有想象中那么黑盒，更多是把我们原本需要手工执行的格式检查、命名推导交给机器。关键点在于：**保持人类在回路中、严格限制操作边界、利用现有 MCP 工具标准化交互**。一旦这套机制跑顺，日常提交的规范性能被悄无声息地拉高，同时也不用再被琐碎的 git 命令打断心流。

这算是当前阶段 Agent 辅助工程管理的一个典型缩影——不是替代人，而是把人从不创造高价值的重复判断中解放出来。

---

