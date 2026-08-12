---
title: 用 OpenClaw Agent 打造 Git 自动化助手：从提交信息到分支管理
feedId: 32764
source: 综合讨论
publishedAt: 2026-08-12
---

# 用 OpenClaw Agent 打造 Git 自动化助手：从提交信息到分支管理

## 背景

每个写过几天代码的人都经历过这样的场景：修了一个小 bug，随手 `git commit -m "fix"`；或者一口气改了 20 个文件，却只能用一句 “update” 敷衍了事。等到三个月后回溯项目历史，满屏都是无意义的提交信息，分支命名也随心所欲，合并冲突时的痛苦指数直线上升。

团队通常会引入 Conventional Commits 等规范，但靠人肉记忆和自律终究不稳定。近一年来，随着 OpenClaw 这类可编程 Agent 框架的成熟，以及 MCP（模型上下文协议）将 Git 操作封装为标准工具，我们有了新的选择：让 AI 助手读懂代码变更，自动生成规范提交信息，甚至根据任务自动管理分支。这篇文章记录我在实际项目中搭建这样一套 Git 自动化流程的过程，包含做法、踩坑记录和可复用的工程建议。

## 问题：手工管理 Git 的低效与不一致

在持续迭代的项目中，手工管理 Git 主要存在三个问题：

1. **提交信息随意**：开发者疲劳或赶工时，常常写出 “fix stuff” 或 “WIP” 这类无意义信息，导致 `git log` 基本废掉。
2. **分支策略执行不到位**：虽然有约定如 `feature/xxx`、`hotfix/xxx`，但手动创建时容易拼错、忘记切分支，或者在错误分支上直接开发。
3. **上下文切换成本**：写完代码还要想一段符合规范的提交信息，打断了心流，尤其是小改动频繁时心理负担明显。

AI 辅助不是要取代开发者的判断，而是把机械化的格式工作和重复操作接管过去，让人类专注于代码逻辑本身。

## 做法：基于 OpenClaw 构建 Git 自动化 Agent

项目环境：OpenClaw 作为 Agent 运行平台，GitLab 私有部署仓库，本地开发机通过 MCP Git Server 暴露 Git 能力给 Agent，LLM 使用 GPT-4o 或 Qwen 等性价比合适的模型。

### 步骤 1：搭建 MCP Git 服务

OpenClaw 的 Agent 通过 MCP 协议调用外部工具。使用 `mcp-server-git` 将本地仓库的 Git 操作包装成 API，并在 OpenClaw 的配置文件中注册：

```yaml
mcp_servers:
  - name: git-local
    command: npx
    args: ["-y", "@anthropic/mcp-server-git", "--repository", "/path/to/repo"]
```

之后 Agent 就能在任务中调用 `git_diff_staged`、`git_commit`、`git_create_branch` 等工具，无需自行拼接命令行。

### 步骤 2：设计 Agent 工作流

我们将自动化拆分为两个独立流程：**提交信息生成** 和 **分支管理**。

#### 提交信息生成流程

1. **触发**：开发者在 IDE 中暂存文件（git add）后，手动运行一个 OpenClaw 快捷指令，或通过文件监控自动触发。
2. **获取变更**：Agent 调用 `git_diff_staged` 获取暂存区的 diff 文本。
3. **LLM 生成信息**：将 diff 和一段精心设计的 prompt 发给大模型，要求输出 Conventional Commits 格式（`type(scope): description`），并限制长度不超过 72 字符的主题行。
4. **人机审核**：生成的候选信息显示在 OpenClaw 的对话窗口中，开发者可以直接确认执行，或编辑后再提交。
5. **执行提交**：Agent 调用 `git_commit` 完成提交，可选自动 push。

Prompt 设计示例核心部分：

```
You are a commit message generator. Analyze the following git diff and produce a commit message following Conventional Commits spec. The subject line must be under 72 characters, imperative mood, no period at the end. Include a brief body if needed.

Diff:
{staged_diff}
```

#### 分支管理流程

我们让 Agent 根据任务类型（issue 标签）自动创建符合规范的分支：

- **触发**：当开发者在项目管理工具中认领一个 issue，并希望开始开发时，通过 OpenClaw 命令触发。
- **读取 issue 信息**：Agent 通过 API 获取 issue 标题、标签（feature/bugfix/hotfix）等。
- **生成分支名**：按 `{type}/{简要描述}` 规则生成，并自动将标题转换为 kebab-case（如 `feature/user-login-refactor`）。
- **创建并切换分支**：Agent 调用 `git_create_branch` 和 `git_checkout`，完成环境切换。
- **记录关联**：Agent 在 issue 评论中自动附上分支名，方便追溯。

这样，每次开始新任务时只需一条指令，分支命名和切换全自动完成，再也不会有 “在 master 上直接写代码” 的心跳时刻。

### 步骤 3：配置与优化

- **审核默认开启**：所有自动生成的提交信息都必须经过人工确认，绝不直接静默提交。避免 AI “脑补”出不可能的改动描述。
- **超长 diff 处理**：当 staged diff 超过模型 token 限制（如 >8000 token）时，Agent 先使用 `git_diff_staged --staged --stat` 获取变动统计，再让 LLM 基于统计信息生成概括性信息，并在 body 中注明 “大量文件变更，细节略”。这虽然丢失细节，但至少比一句 “update” 好。
- **回滚保护**：自动提交前记录当前 commit hash，一旦开发者反馈错误，Agent 可以快速执行 `git reset --soft HEAD~1` 撤消。

## 踩坑记录

在实际使用中，我们踩了几个非显而易见的坑：

1. **大 diff 导致幻觉**：即使模型生成了详细描述，偶尔也会张冠李戴，把 A 函数的改动说成 B 函数。解决方案是强制 Agent 在生成信息后，用 diff 中的函数名做一次简单验证，发现不匹配就标红提醒审核。
2. **git 操作权限泄漏**：MCP Server 以本地用户权限运行，意味着 Agent 拥有和你一样的 Git 权限。千万不能把 `--force` 等危险选项暴露给 LLM。我们在工具定义中明确移除了 `force` 参数。
3. **分支命名冲突**：多人同时基于相似 issue 创建分支时可能重名。Agent 在创建前会检查 `git branch --list`，如果存在同名分支自动追加随机四位后缀，并提示开发者。
4. **上下文污染**：有些开发者习惯本地有多个 WIP 暂存，自动提交会把这些半成品代码也提交进去。所以我们强制要求仅处理 **已暂存且与上次提交相比有实质性差异的变更**，工作区修改未暂存则忽略。

## 可复用建议

结合这段实践，总结几条工程化落地建议：

- **保持半自动化**：AI 负责“拟稿”，人类负责“签发”。这是安全与效率的平衡点，也符合 Git 操作的改错成本特性。
- **规范先行，模板固化**：在 prompt 中内嵌团队约定（如 scope 可选值、必须引用的 issue 编号），避免每次生成风格漂移。
- **小步提交**：鼓励开发者频繁暂存小块逻辑变更，这样 AI 生成的 message 更精确，diff 也不易超 token。
- **入职机器搭档**：把 OpenClaw 的这条自动化流程写入团队 onboarding 文档，新人第一天就能用上，规范落地零推广成本。
- **监控与反馈**：记录每次自动生成的 message 是否有被人工修改，统计采纳率，持续迭代 prompt。

## 总结

Git 自动化并不是要打造一个“无人的版本控制”，而是把高度重复、规则明确的脏活累活移交给 AI 助手。通过 OpenClaw + MCP Git 的组合，我们实现了自动化的提交信息生成和分支管理，在保证安全审核的前提下，使团队的提交历史清晰度明显提升，分支策略执行率几乎达到 100%。对于已经在使用 OpenClaw 或类似 Agent 平台的团队，这套方案很轻量，两小时内即可搭建并投入试用。有兴趣的读者可以从简化版本开始，先用 AI 生成 message 手动确认，逐步扩展到分支管理，体验一下“有 AI 帮你记 Git 规范”的轻松感。

---

