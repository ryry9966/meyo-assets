---
title: 给 Agent 写一份 USER.md：让自动化助手真正了解你的偏好与上下文
feedId: 31971
source: 综合讨论
publishedAt: 2026-08-07
---

## 背景：Agent 为什么需要“你是谁”的定义？

在 OpenClaw、自定义 Agent、MCP 工具链和自动化脚本的实践中，越来越多的人开始让 AI 助手执行跨项目操作、生成配置文件、发送消息甚至操作本地文件系统。但无论是用 Claude Code、Codex CLI 还是自己搭建的 LangChain 链，Agent 往往会遇到同一个瓶颈：**它不知道你的习惯、设备名、常用路径、项目结构偏好，甚至你的缩写风格。**

你可能每次都要在 prompt 里重复：“我习惯用 `~/workspace` 放项目，请用两个空格缩进，不要生成 `.env.example`，数据库端口默认是 5433。” 这很烦，也很容易遗漏。于是社区出现了一种朴素但有效的解法：在项目或用户目录里放一个 `USER.md`，让 Agent 每次启动时自动加载，当作永久性的“用户手册”。

这个做法最早来自一些 CLI Agent 工具的内置机制（如 OpenClaw 会尝试读取 `~/.openclaw/USER.md`），但现在完全可以用在任何读文件链路里，甚至通过 MCP 资源或系统提示注入。

## 问题：没有 USER.md 时，你踩过哪些坑？

在没有持久用户上下文的情况下，Agent 的行为通常表现为：

1. **重复询问**：每次新会话都要你重新描述环境，打断心流。
2. **错误假设**：生成你从不用的目录（如 `~/Documents/dev`）或使用错误的包管理器（你明明用 pnpm，它偏写 npm）。
3. **安全风险**：把密钥路径写死在生成的配置里，或者在日志中不经意输出你不愿暴露的目录结构（如含有真实姓名的路径）。
4. **风格混乱**：今天生成 snake_case，明天变 camelCase，后天建议用 class 组件而你只用函数式。
5. **无法利用深度个人优化**：比如你知道某个内部服务总是在 `http://192.168.1.100:8080`，Agent 不知道，每次都要你提供。

这些看似微小，但在高强度使用 Agent 做实际工作时，会显著增加检查成本。一份结构良好的 `USER.md` 可以直接消除大部分摩擦。

## 做法：怎么编写并接入你的 USER.md

### 1. 文件位置与加载策略

优先选择能被 Agent 自动发现的位置。以 OpenClaw 为例：

- 全局配置：`~/.openclaw/USER.md`
- 项目级覆盖：`<project>/.openclaw/USER.md`

你可以用环境变量 `OPENCLAW_USER_MD` 指定路径，或直接在 prompt template 中通过 `${USER_MD}` 注入。

对于其它工具（如自建的 MCP 服务或 Python Agent），可以在入口文件里读取：

```python
import os, pathlib
user_md_path = os.getenv("USER_MD_PATH", pathlib.Path.home() / ".config/agent/USER.md")
if user_md_path.exists():
    system_prompt += "\n\n## User context\n" + user_md_path.read_text()
```

这样无论是通过 CLI 还是 MCP Server 调用的 Agent，都能拿到这份上下文。

### 2. USER.md 应该写什么？（实战模板）

我自己的 `~/.openclaw/USER.md` 结构如下，你可以直接复制骨架：

```markdown
# USER.md - Personal context for AI assistants

## Identity & Roles
- Name: Alex (prefers informal tone)
- Timezone: Asia/Shanghai (UTC+8)
- Languages: replies in Chinese when context is Chinese, otherwise English.

## Development Environment
- OS: macOS 14 / Debian 12 on servers
- Shell: zsh with oh-my-zsh
- Preferred editor: VS Code (insiders), `code` command available
- Default package manager: pnpm (do NOT use npm or yarn)
- Node version: 22 LTS via fnm
- Python: 3.12 via pyenv, use `uv` for package management

## Project defaults
- Web frameworks: Next.js (App Router), FastAPI
- CSS: Tailwind CSS, avoid CSS modules unless specified
- Testing: Vitest (frontend), pytest (backend)
- Linting/formatting: ESLint + Prettier, always semicolons
- Environment variable style: never commit `.env`, always use `.env.example` with only keys (no values)

## Directory conventions
- All projects live in `~/workspace/<domain>/<project>`
- Open-source clones under `~/playground/`
- Scratch/test code in `~/lab/`

## Preferences & Habits
- Prefer explicit error handling, never swallow exceptions
- Code comments in English, commit messages in English
- Use conventional commits: `feat:`, `fix:`, `chore:` etc.
- When generating shell commands, assume GNU coreutils and use long options for clarity
- Never generate decorative comments like "// This is a function"

## Services & Endpoints
- Local LLM: Ollama at `http://localhost:11434`
- Internal registry: `http://192.168.3.10:4873`
- Dev database: postgresql://dev@localhost:5433/devdb (no password)

## Constraints
- NEVER expose API keys or real credentials in generated output
- Do NOT suggest creating docker-compose files unless explicitly asked
- Do NOT mention user's real name in examples or placeholder values
```

### 3. 注入时的注意事项

- **隐私边界**：不要把完整的真实姓名、公司名、密钥放进 USER.md。如果 Agent 运行在云端，这份文件会被发送到 API。只放相对安全的偏好信息。
- **格式**：用 Markdown 分区标题，Agent 解析会更稳定。避免多余的无序符号。
- **长度控制**：建议不超过 200 行。过长的上下文会稀释有效指令，甚至增加 token 消耗。

## 踩坑点：这些坑我替你趟过了

### 坑1：自动加载失败时没有回退提示
有些工具在文件不存在时，会悄悄跳过，不会告诉你。结果你写好了 USER.md 放在自以为正确的位置，但 Agent 根本没读。**解法**：在第一次使用新 Agent 时，显式要求它打印当前加载的 USER.md 路径和内容摘要，确认生效。

### 坑2：项目中同时存在多个用户描述文件
比如项目里有 `CLAUDE.md`、`AGENTS.md`、`USER.md`，Agent 可能都读，导致指令冲突。**解法**：统一为 `USER.md` 或 `CONTEXT.md`，并在工具配置中明确指定只读取一个，或者设置优先级规则。

### 坑3：内容过时却不自知
半年后你的默认 Node 版本已经从 18 变成了 22，但 USER.md 还写着 18。Agent 会坚持用旧版配置，你却在排查为什么生成的环境不一致。**解法**：在版本控制中管理 USER.md，或者写一个 `update_agent_context` 的快捷命令，定期提醒自己检查。

### 坑4：`USER.md` 中写了不兼容的系统级假设
比如写了“默认使用 Docker 部署”，但某个轻量脚本环境根本没装 Docker，Agent 仍会坚持生成 docker-compose 文件。**解法**：用条件描述，如“在涉及部署时优先使用 Docker（若在支持 Docker 的环境中）”。

## 可复用建议：让 USER.md 成为基础设施

1. **把 USER.md 纳入 dotfiles 管理**  
   用 chezmoi、yadm 或 git bare repo 同步到所有设备，保证多机器一致。

2. **按域拆分**  
   如果工具支持，可以让 `USER.md` 只放用户信息，再配合项目级的 `PROJECT.md` 放团队约定、技术栈等。目前 OpenClaw 可通过合并多个文件实现（如 `SYSTEM.md` + `USER.md`）。

3. **利用 MCP 资源提供动态上下文**  
   更进一步，你可以让 MCP Server 提供一个 `user://preferences` 资源，里面不仅包含静态文本，还可以注入当前 git 用户、操作系统、可用网络接口等动态信息。这样 Agent 不用猜测。

4. **与团队共建 AI 准备文档**  
   除了个人 `USER.md`，在项目仓库根目录放一份 `CLAUDE.md` 或 `AIDEV.md`，约定分支命名、编译命令、测试要求等。这会让团队内部的 Agent 体验高度统一。

## 总结

`USER.md` 的做法看似简单粗暴，但它解决了 Agent 自动化中最被低估的一环：**可预测性**。当你的 Agent 不再需要猜测“你是用 pnpm 还是 npm”、“文件放 `~/dev` 还是 `~/workspace`”时，指令的准确度和执行速度会有肉眼可见的提升。更关键的是，它把开发者的隐性知识显性化，让你从不断校正 AI 的角色中解放出来，专注于真正需要决策的地方。

如果你已经在用 OpenClaw、Claude Code 或类似的 CLI Agent 工具，今天就可以花 15 分钟写一份 `USER.md`，然后观察接下来一周的交互变化。你会发现，Agent 终于开始“懂你”了。

---

