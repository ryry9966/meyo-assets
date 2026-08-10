---
title: 从自然语言到 commit：用 MCP 搭一套可控的 Git 自动化助手
feedId: 32481
source: 综合讨论
publishedAt: 2026-08-11
---

## 背景：你已经在用 AI 写代码，但还在手动 git commit？

不少开发者已经在 IDE 或 Agent 里让大模型帮忙生成代码、修复 bug，甚至写 PR 描述。然而一到 `git add`、`git commit`、分支切换，还是得回到终端敲命令，或者手写 commit message —— 不但打断心流，也容易因为惰性写出 “fix bug”“update” 这种没有信息量的提交信息。

更麻烦的是，在一个多人协作的工程里，commit message 格式、分支命名规则往往有严格的 lint 约束（如 conventional commits），对新手或者赶进度的人而言，这层约束比代码本身更“反人性”。

于是很自然会想：能不能把我日常跟 Agent 的交互直接延伸到 Git 操作上？比如我说一句 “把刚才对登录模块的改动做一次 feat 类型的提交”，Agent 就能干活，并且自动遵守项目的提交规范。

## 问题拆解：安全、权限与规范的平衡

要让 AI 管理 Git，并不是写个 `subprocess.run("git commit -m 'xxx'")` 那么简单。至少要解决三个层面的问题：

1. **权限边界**：只能操作指定仓库，不能到处乱跑。
2. **操作安全**：不能静默 force push、不能随意修改受保护分支。
3. **规范落地**：commit message 必须符合项目规则（例如 `feat(scope): subject`），而且生成逻辑要有确定性，不能每次随缘。

这类需求在 MCP（Model Context Protocol）出现之后变得非常自然——把 Git 能力封装成一个 MCP server，让 OpenClaw 这类 Agent 通过标准化的工具调用去使用。于是控制权回到我们手里：可以限制 server 暴露哪些 Git 子命令、在哪个目录执行、是否要求二次确认。

## 实践：搭建 Git MCP server + Agent 工具链

下面是一个可复现的搭建步骤，假设你已经在用支持 MCP 的 Agent（如 OpenClaw 客户端）。

### 1. 启动 Git MCP server

从社区选用一个实现，例如 `mcp-server-git`。该 server 通常基于 Python，默认暴露 `git_status`、`git_diff_unstaged`、`git_add`、`git_commit`、`git_branch`、`git_checkout` 等工具。

```bash
pip install mcp-server-git
mcp-server-git --repository /path/to/your/repo
```

启动时务必指定 `--repository` 参数，把操作范围锁死在一个目录内。如果你需要管理多个仓库，可以为每个仓库启动独立的 server 实例，避免权限泄漏。

### 2. 在 Agent 配置中注册工具

以 OpenClaw 为例，在 `agents/config.yaml`（或类似位置）添加：

```yaml
tools:
  - mcp:
      command: "mcp-server-git"
      args: ["--repository", "/home/user/project"]
```

重启 Agent 后，对话界面就能识别这些 Git 工具。此时你可以用自然语言下达指令：

> “查看当前工作区改动，只把 src/auth 下的文件 stage，然后用 conventional commit 格式提交，范围是 auth，类型是 feat。”

Agent 会依次调用 `git_diff_unstaged`、条件性 `git_add`，然后生成符合格式的 commit message 并调用 `git_commit`。

### 3. 用系统提示词固化规范

单纯靠用户每次手动要求“按 conventional commit 格式”是不够稳定的。更好的做法是在 Agent 的 system prompt 中加入强约束：

```
当执行 git commit 时，你必须：
- 使用 conventional commits 格式：type(scope): subject
- body 可选，但如果有破坏性变更必须用 BREAKING CHANGE 标注
- 绝不使用 -m 直接拼接多行，使用临时文件编辑 commit message
- 提交前必须展示 commit message 给用户确认（除非用户明确说 --yes）
```

这样一来，Agent 的提交行为就有了确定性，不只是“概率正确”。

## 踩坑记录

1. **路径穿越**：如果 server 没有严格控制 `repository` 参数，Agent 可能通过 `../../` 操作到无关目录。一定要使用支持路径白名单的实现，或者用容器/沙盒再隔一层。
2. **自动 stage 太多**：AI 有时会把不相关的文件也 `git add`，特别是生成的临时文件。可以用 `.gitignore` 配合 `pre-commit` hook 做兜底拦截。
3. **commit message 语言不一致**：如果你同时使用中英文，Agent 可能生成混合语言的 message。建议在 system prompt 中指定语言，或在 server 层加入简单的 lint 检查。
4. **敏感文件泄漏**：`.env`、密钥文件可能出现在 diff 中。建议在 `git_diff` 工具描述里加入提示，让 Agent 检查是否包含敏感信息，必要时拒绝展示完整内容。

## 可复用建议

- **小步验证**：先用只读工具（`git_status`、`git_diff`）试跑，确认 Agent 能正确理解仓库状态，再逐步开放写操作。
- **二次确认机制**：除非是 CI 场景，否则 commit/push 之前最好都要求用户确认。可以在 Agent 配置里设定 `require_confirmation: ["git_commit", "git_push"]`。
- **与现有 hook 协同**：把 AI 提交当成普通开发者的提交，让 `commitlint`、`pre-commit` 这些 hook 工作起来。多次审查总比没有好。
- **记录日志**：让 Agent 将操作日志输出到一个固定文件，方便回溯谁让 AI 干了什么。

## 总结

Git 自动化不是用 AI 取代开发者对仓库的控制，而是把那些重复、格式敏感的操作下沉给工具。基于 MCP 的方案最大优势在于“可控”：你可以精确限定操作范围、插入确认节点、复用已有的工程规范。一旦这套体系跑顺，你会发现提交信息质量的提升是立竿见影的——而你自己只要花更多时间在架构和代码上，而不是和 commitlint 斗智斗勇。

---

