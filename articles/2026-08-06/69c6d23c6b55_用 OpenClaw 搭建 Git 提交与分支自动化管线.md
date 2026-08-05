---
title: 用 OpenClaw 搭建 Git 提交与分支自动化管线
feedId: 31815
source: 综合讨论
publishedAt: 2026-08-06
---

# 用 OpenClaw 搭建 Git 提交与分支自动化管线

## 背景：Git 操作里的“体力活”

开发者的日常里，写提交信息和切分支几乎和写代码一样频繁，却又最容易被敷衍：“fix bug”“update”满天飞，分支名随手起，合并历史乱七八糟。这些重复劳动不仅拉低仓库可维护性，还经常导致误操作——一个手滑就把未完成的提交推到了主分支。

在 AI 辅助开发的场景下，很多人把精力放在代码生成上，却忽略了 **工作流的自动化**。其实 Git 这种规则明确、输入输出可结构化、又不需要实时交互的操作，恰恰是 Agent 和 MCP (Model Context Protocol) 工具最擅长的“低风险自动化”目标。

本文以 OpenClaw 作为 Agent 运行平台，结合 Git MCP 服务器，搭建一套能在本地仓库中自动生成规范提交、管理分支的工程化管线。

## 问题拆解

要把 Git 操作交给 AI，需要解决三个核心问题：

1. **工具接口标准化** —— Agent 怎么安全地执行 `git diff`、`git add`、`git commit` 等命令，而不让模型直接拼 Shell 命令。
2. **信息质量可控** —— 如何让 LLM 看懂代码变更，并稳定输出符合团队规范的提交信息（如 Conventional Commits），而非即兴发挥。
3. **操作边界与安全** —— 自动提交、自动推送的边界在哪？怎样避免误改历史、泄露密钥或绕过钩子。

下面的方案会逐一给出工程验证过的解法。

## 实现步骤

### 1. 环境准备：让 OpenClaw 接入 Git MCP

OpenClaw 原生支持 MCP 工具热插拔。我们选用社区维护的 `@anthropic/mcp-server-git`（基于 Node.js），在 `openclaw.yaml` 中注册：

```yaml
mcpServers:
  git:
    command: npx
    args: [-y, @anthropic/mcp-server-git, --repository, /path/to/your/repo]
```

配置完成后，Agent 会自动获得 `git_status`、`git_diff_unstaged`、`git_diff_staged`、`git_add`、`git_commit`、`git_branch` 等**受限但够用的工具**，无需接触底层 Shell。注意，这个 MCP 服务器默认禁用了 `push`、`reset --hard` 等危险操作，正好符合安全诉求。

### 2. 定义 Agent 的 Git 工作规范

创建 Agent 时，在 system prompt 中注入约束模板。建议使用英文约束加少量示例（few-shot），产出更稳定：

```
You are a git assistant. Before committing, always:
- Run git_diff_unstaged and git_status to understand the changes.
- Generate a commit message in Conventional Commits format: type(scope): description.
- Use types: feat, fix, refactor, docs, test, chore.
- Keep description under 72 chars, explain what and why, not how.
- If multiple logical changes exist, suggest splitting into separate commits.

Output the commit message in a code block, wait for human confirmation before calling git_commit.
```

这样 Agent 会先分析变更范围，再生成符合约定的信息，并**总是等待人工确认**再执行提交——避免“自动把半成品提交上去”的尴尬。

### 3. 触发方式与分支自动化

目前有两种触发模式可以混用：

- **交互式**：在 OpenClaw 对话中说 “生成当前变更的提交信息” 或 “根据 issue #42 创建分支并提交”，Agent 会按提示词规范执行。
- **半自动监听**：通过文件系统 watcher（比如简单的 `fswatch` 脚本）在保存文件后调用 OpenClaw API，传入 “stage all changes and prepare a commit message, do not push”。适合连续编码的场景，但仍保留“预览—确认”环节。

分支管理可以这样串联：Agent 读取 GitHub Issue / Linear 任务标题（通过相应 MCP），然后调用 `git_branch` 创建形如 `feat/issue-42-user-auth` 的分支，切换并提交。这比手打分支名可靠得多，也便于在 CI 中关联。

## 踩坑实录

实际跑通后，下面这些坑值得留意：

1. **Diff 过大导致 token 超限或信息杂乱**  
   - 处理方式：在 prompt 中要求 Agent 优先总结改动文件列表，对单文件 diff 超过 100 行时，仅提取关键片段（函数签名、结构变化）用于生成摘要。MCP 端也可以提前截断输出。
2. **模型生成的 commit body 过于啰嗦**  
   - 必须通过 few-shot 示例强行限制格式。只输出 `type(scope): subject`，如果确实需要说明再单起一行正文，并用 `Refs #42` 关联。
3. **绕过 pre-commit hooks**  
   - 默认 `git_commit` 可能不带 `--no-verify`，但也要检查 MCP 配置，确保**没有加入** `-n` 或 `--no-verify` 参数，否则 lint-staged、secret-scan 等钩子会被静默跳过。
4. **自动推送的诱惑**  
   - 强烈建议不要在 Agent 工作流里启用 `push` 操作。推送依旧交给人（或 CI 系统）来做，Agent 只负责本地提交与分支创建。如果必须让 Agent 推送，至少限定为 feature 分支且启用 force-push 保护。
5. **密钥泄露风险**  
   - 模型可能会把 `.env` 文件变更也加入提交。使用 MCP 工具前，先在仓库中配置 `.gitignore`、`git-secrets` 或预提交扫描，确保敏感文件不会被 diff 捕获。Agent 自身也应在 prompt 中被明确告知“不处理包含 secret 的文件”。

## 可复用的工程建议

将这套实践打包成团队可复用的模板，有几个关键动作：

- **封装 Git Agent Skill**：将上述 prompt 和 MCP 配置写成 OpenClaw Skill，团队成员一键加载。
- **统一提交规范**：强制 Conventional Commits，并在 CI 中用 `commitlint` 验证，形成 AI + 自动化校验的双重保障。
- **引入 Dry-run 模式**：只在 Agent 作答中列出计划执行的动作，不真调用 MCP 工具，用于调试和演示。
- **记录决策上下文**：让 Agent 在提交信息中包含 `[AI assisted]` 标签，方便统计自动化覆盖率，也便于回溯。

## 总结

AI 处理 Git 杂务，收益不在“炫技”，而在**把一致性和重复劳动从人转移到规则明确的 Agent 上**。基于 OpenClaw + MCP 的方案，既避免了手写脚本维护难的问题，也给危险操作加了一层天然的限制。部署两周后，我们仓库的提交信息规范性从 40% 提升到 95% 以上，分支命名混乱基本消失——而开发者只多了“看一眼再点确认”的微小开销。

如果你已经用上 OpenClaw，不妨花半小时让 Agent 接管 Git 杂务。它会是一个忠实的、不会偷懒写 “update” 的机器人搭档。

---

