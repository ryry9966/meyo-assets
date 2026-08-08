---
title: AI 助手的 USER.md：让 Agent 真正了解你是谁
feedId: 32201
source: 综合讨论
publishedAt: 2026-08-09
---

# AI 助手的 USER.md：让 Agent 真正了解你是谁

## 背景

OpenClaw、MCP、插件生态让 AI 助手具备了读写本地文件、执行命令、调用外部服务的能力，Agent 正在从“对话机器”变成“数字执行体”。但是，要让 Agent 按下正确的组合键、选择你偏好的工具链、理解你惯用的术语缩写，仍需要大量手动提示。每次新会话都要重述一遍“我是后端开发，用 Rust+Neovim，别碰我的 docker-compose.dev.yml”，不仅低效，还容易遗漏。

能不能让 Agent 在每一次启动、每一次执行关键动作之前，主动读一份关于“你是谁”的描述文件，从而建立上下文基线？这就是 **USER.md** 的思路——一份类似项目里的 `README.md`，但描述的不是项目，而是使用者本身。

## 问题拆解

目前给 Agent 注入用户特征的常见做法有几种：

1. **系统提示词硬编码**：把偏好写死在 `systemInstruction` 或前端配置里。缺点是不容易动态更新，且随着描述增多会挤占上下文窗口。
2. **每次粘贴偏好**：在提问框里附上一段用户档案文本。缺点是不可复用、易遗忘。
3. **用 Prompt 模板注入变量**：如 `{{user.os}}` 等，但覆盖面有限，难以描述复杂的工具链习惯。

上述方法都没有利用 OpenClaw / MCP 生态里最朴素的能力：**文件系统访问**。如果你有能力让 Agent 读取本地文件，那为什么不干脆维护一个用户档案文件，让 Agent 在有需要的时候直接读取？

## 做法 / 步骤

### 1. 确定存储位置与格式

个人建议使用 `~/.openclaw/USER.md` 或项目根目录的 `AI_USER.md`（视作用域而定）。文件格式为 Markdown，便于人类编辑和 Agent 解析，同时也方便嵌入到其他工具链（如 GitHub，Obsidian）。

最简单的结构示例：

```markdown
# USER PROFILE

## Identity
- Name: Alex
- Role: Backend / Infra Engineer
- Pronouns: they/them

## Environment
- OS: macOS 15 + Linux (Ubuntu 24.04)
- Shell: zsh, tmux
- Editor: Neovim (with LazyVim), VS Code for debugging
- Terminal: iTerm2 (macOS), Alacritty (Linux)

## Tech Preferences
- Languages: Rust, Go, Python (only for scripts)
- Infrastructure: Docker (compose v2), Terraform, AWS CLI
- Version control: Git, GitHub CLI, pre-commit
- Linters: clippy, golangci-lint, ruff

## Project-specific rules
- NEVER delete docker-compose.dev.yml
- Use `just` command runner when available
- Before modifying databases, always show plan for review

## Terminology
- "prod-like" = staging environment
- "green" = all tests passing
- "SVC" = services directory under monorepo
```

要点是**结构化、克制**。Agent 不需要知道你昨晚吃了什么，它只需要能正确推断出你想用 `cargo check` 而不是 `npm run lint`。

### 2. 让 Agent 能够主动读取

在 OpenClaw 的配置规则（如 `claw.md`）或 MCP 服务器的定义里，应当设计一条“检查 USER.md 是否存在并读取”的策略。例如，在 `systemInstruction` 中加入：

```
Before executing any task that involves file editing, command execution, or code generation, check if ~/.openclaw/USER.md exists.
If it does, read the file and adapt your behavior according to the preferences defined there.
```

如果是基于 MCP 的架构，可以通过 `resources/list` 暴露这个文件资源，或者使用 `fs` 服务器的 `read` 工具，让 Agent 在决策环节自动调用。关键在于，**不要等到用户提示“请读我的 profile”才去读**——应当成为 Agent 内置的、有条件的自动化行为。

### 3. 将 USER.md 纳入上下文管理

为避免每次会话都浪费 token 去重读一遍，可以设计一个轻量级的缓存机制。比如，Agent 启动时读取一次，将关键字段解析后存储在会话状态对象里，后续只需按需检索。若文件修改时间发生变化，再触发刷新。

这一层不同框架实现方式不同。但核心原则是：USER.md 的内容应该被转换为结构化的内部表示（例如 JSON），而不是每次都把原始 Markdown 塞进提示词里。

### 4. 测试与验证

写一份简单的测试用例来确认 Agent 是否正确读取并应用了你的偏好：

- 给出一条模糊的指令：“帮我创建一个新的快速脚本，用来重启本地开发环境。”
- 观察 Agent 是否使用了 `~/.openclaw/USER.md` 中定义的 Shell（zsh）、脚本语言（Python）、是否避免触碰 `docker-compose.dev.yml`。

如果不生效，检查：
- 路径是否正确，是否对 Agent 进程可读。
- 是否在规则中声明了“始终读取”。
- 文件编码（UTF-8），没有 BOM 头。

## 踩坑点

- **路径歧义**：如果同时存在全局 `~/.openclaw/USER.md` 和项目级 `AI_USER.md`，Agent 该读哪个？需要定义优先级。建议项目级覆盖全局，并允许规则中指定路径。
- **信息泄露**：该文件可能被包含在共享会话、自动提交的日志中。不要在 USER.md 里放 token、密码、客户名称等敏感信息。
- **内容膨胀**：不小心就会写成个人维基。建议限制在**对任务执行有直接影响**的信息，约束在 100 行以内。定期修剪。
- **Agent 过度依赖**：有时 Agent 会机械地引用 USER.md 的内容，反而忽略当前会话的显式指令。需要让“显式覆盖”原则生效，即用户在当前对话中的新要求，应当覆盖 USER.md 中的一般偏好。
- **文件权限**：在 Linux 或多用户环境下，如果 Agent 进程以不同用户运行，可能无法读取家目录下的文件。考虑使用 `/etc/openclaw/user-profile.md` 或环境变量指定路径。

## 可复用建议

1. **模板化**：为团队提供模板，包含常用字段（OS、Editor、Shell、语言、重要规则），降低登记成本。可存放在共享仓库中。
2. **与 onboarding 相结合**：新成员加入项目时，填写 `AI_USER.md` 成为开发环境设置的一环。就像 `.editorconfig` 统一了编辑器风格，USER.md 统一了 Agent 的交互范围。
3. **与 MCP 服务器联动**：可以编写一个专用的 MCP 服务器——“user-profile”，它不仅提供静态 USER.md，还能动态获取当前环境信息（当前 git 分支，操作系统版本），进一步减少手动描述的工作量。
4. **保持版本控制**：对于项目级的 `AI_USER.md`，纳入 Git 管理并做 code review，避免引入有破坏性的规则。

## 总结

USER.md 是一个极简但有效的手段，让 AI 助手在本地化、个性化执行时少犯低级错误。它不依赖任何高级模型能力，只靠“文件读取”这个最基础的操作，就能大幅减少每次对话中重复的解释工作，同时带来更可预测的助手行为。

在一个 Agent 能力快速膨胀的时代，人们对提示词工程的讨论热烈，却容易忽略一个朴素的道理：**如果你想让一个执行体真正理解你，不如先诚实地写下来，然后让它去读。** USER.md 就是那面镜子。

---

