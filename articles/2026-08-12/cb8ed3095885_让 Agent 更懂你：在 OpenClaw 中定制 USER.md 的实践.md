---
title: 让 Agent 更懂你：在 OpenClaw 中定制 USER.md 的实践
feedId: 32778
source: 综合讨论
publishedAt: 2026-08-12
---

# 让 Agent 更懂你：在 OpenClaw 中定制 USER.md 的实践

## 背景：Agent 很强，但还不够懂你

OpenClaw 这类工具把 Claude 接入了本地环境，能读文件、跑命令、操作浏览器，已经非常接近一个真正的“工程搭档”。但反复用上几天后你会发现一个问题：每次让它帮忙重构代码、写测试、或者启动一个本地服务，都要先手动把项目路径、端口号、代码风格、常用命令交代一遍。你不说，它就猜；猜错了，再来一轮。

这不是模型能力问题，而是 Agent 缺少一份稳定、持久的“用户画像”。系统提示（system prompt）只能定义通用行为，没法描述你个人的开发习惯、工具偏好和项目的局部知识。于是每次对话都在重复建立上下文，既浪费 token，又容易遗漏关键约束。

## 问题：口头交代 ≠ 工程化

常见的补救方式有三种：

1. 每次对话开头手打一段背景描述 → 低效、易忘。
2. 把个人偏好写进 OpenClaw 的系统提示里 → 难以维护，不适合项目间切换。
3. 在对话中口头“调教” Agent 记住你 → 它没有长期记忆，换个会话就清零。

这些方式都缺乏**版本化、模块化和可移植性**。事实上，Agent 需要一个像 `README` 之于项目那样的东西——但它的读者不是人，而是 Agent 自身。这就是 `USER.md`。

## 做法：把个人上下文写进 USER.md

核心思路很简单：创建一个 Markdown 文件，在 OpenClaw 启动时将其内容注入到当前会话的上下文中，让 Agent 在每次对话开始时就能读到你的“使用说明书”。

### 步骤 1：确定文件存放位置

推荐两种方式，按需选择：

- **全局位置**：`~/.openclaw/USER.md`  
  适合存放与你个人强相关的通用信息（如编辑器、shell、默认技术栈等），所有项目共享。
- **项目级位置**：`<project_root>/.openclaw/USER.md`  
  存放当前项目特有信息（目录结构、构建命令、环境变量命名规则等）。

OpenClaw 支持在配置文件中声明这些注入源。我习惯同时启用两个，让项目级内容自动覆盖或补充全局配置。

### 步骤 2：编写 USER.md 的内容

写 `USER.md` 要遵循一个原则：**只写 Agent 需要知道且不能轻易推断的信息**。不要把它写成自传，保持精炼，减少 token 占用。一个典型的工程化模板如下：

```markdown
# User Profile

## Identity
- Name: Alex
- Role: full-stack developer (Rust + React)
- OS: macOS / Debian on servers
- Preferred shell: zsh, with oh-my-zsh

## Global Preferences
- Code style: single quotes, no semicolons in JS/TS, 2-space indent
- Linter: ESLint with Airbnb config, Prettier
- Test runner: vitest for frontend, cargo test for backend
- Commit convention: conventional commits, no emojis
- LLM interaction style: be concise, show only relevant diffs

## Environment
- Default node version: 20 LTS via nvm
- Rust toolchain: stable, installed via rustup
- Package manager: pnpm (do not suggest npm or yarn)
- Local AI endpoint: http://localhost:11434 (Ollama)

## Common Tasks
- Start frontend dev server: `cd frontend && pnpm dev` (runs on 5173)
- Start backend: `cargo run` in ./backend, binds to 3000
- Run all tests: `pnpm test` (frontend), `cargo test` (backend)
- Lint: `pnpm lint` and `cargo clippy`

## Project-Specific Notes (work-project/.openclaw/USER.md)
- This project uses shadcn/ui with Tailwind, do not suggest other CSS frameworks
- Database URL is read from DATABASE_URL env var, never hardcode
- Auth via NextAuth credentials provider, custom pages in /app/auth
```

对于项目文件，重点描述 API 路径约定、环境变量、特殊依赖等。内容要**可执行**：Agent 看到 `start frontend` 就知道具体命令，不再需要你解释。

### 步骤 3：让 OpenClaw 加载 USER.md

OpenClaw 通常通过 `claude.yaml` 或 `mcp.json` 配置文件控制上下文注入。以全局配置为例，在 `~/.openclaw/claude.yaml` 中添加：

```yaml
context:
  include_files:
    - path: ~/.openclaw/USER.md
      enabled: true
```

如果同时想加载项目文件，可以在项目级配置中追加路径，并确保合并策略为 `merge`：

```yaml
context:
  include_files:
    - path: .openclaw/USER.md
      enabled: true
```

某些变体支持通过 MCP server 动态提供上下文，例如使用 `file-reader` server 在会话开始时自动读取 `USER.md`。无论哪种方式，目标都是**无感注入**：你启动会话，Agent 就已经知道你是谁。

### 步骤 4：验证效果

重新启动一次 OpenClaw 会话，发一条不包含额外说明的测试指令，例如：

- “帮我写一个单元测试。”
- “启动开发服务器。”

观察 Agent 是否直接使用了 `USER.md` 中的偏好（比如用 vitest 而不是 jest，用 `pnpm dev` 而不是 `npm run dev`）。如果 Agent 仍然询问细节，检查：文件路径是否正确、配置是否被加载、文件格式是否有导致解析错误的字符（如 Tab 而非空格）。

## 踩坑点

1. **Token 饥饿**  
   `USER.md` 太长会挤占实际任务空间。建议全局文件控制在 500 token 以内，项目文件 300 token 以内。必要时用 `#` 分区，让 Agent 按需检索——但目前多数实现是全量注入，所以必须克制。

2. **隐私泄漏**  
   不要把 token、密码等直接写进文件。`USER.md` 通常是明文、会被版本控制，也可能随 OpenClaw 的调试日志暴露。敏感信息用环境变量占位符，如 `DATABASE_URL from env`。

3. **更新不生效**  
   修改 `USER.md` 后，当前会话可能缓存了旧版本。务必重启会话或清理 OpenClaw 的上下文缓存。确认加载顺序：启动时会打印加载了哪些文件。

4. **与系统提示冲突**  
   系统提示优先级一般高于 `USER.md`。如果系统提示明确要求某种代码风格，Agent 会遵循系统提示而忽略你的个人设置。解决方式是尽量让 `USER.md` 补充而非覆盖：写明“当未指定时，默认为……”。

5. **跨项目污染**  
   全局 `USER.md` 中的偏好可能不适用于全部项目。例如全局写死了 React，但某个项目用的是 Vue。此时项目级 `USER.md` 应明确覆盖对应项，并通过注释提醒 Agent：“Project overrides global: use Vue + Pinia instead of React.”

## 可复用建议

- **模块化拆分**：按关注点拆成 `USER.base.md`（身份、全局偏好）、`USER.project.md`（项目特性），通过脚本在启动时合并为一个文件注入。这样多个项目或多人可以复用基础画像。
- **利用 MCP 动态获取**：如果 OpenClaw 环境支持 MCP，可以编写一个简单的 MCP server，接收 `--profile` 参数并按需加载不同 persona（比如“review模式”、“debug模式”），避免一个 USER 文件处理所有场景。
- **版本控制与同步**：将 `USER.md` 纳入 dotfiles 仓库，通过符号链接或 chezmoi 同步到多台机器。这样在任何环境打开 Agent，使用习惯都是一致的。
- **定期修剪**：每隔一个月检查一次文件，删掉不再使用的命令、废弃的工具链，防止 token 悄然膨胀。
- **团队共享基础画像**：对于团队内部常用的工具配置（如统一 linter 规则），可以抽取一份 `TEAM.base.md`，个人在此基础上补充。但注意不要强制覆盖个人偏好，留给开发者可定制的空间。

## 总结

`USER.md` 不是一个新概念，它本质上是给 Agent 的一份“静态记忆”。投入一小时整理自己的常用偏好，能在后续每一次对话中为自己省下反复交代的时间。它不解决模型的推理能力问题，但能大幅降低交流摩擦——让 Agent 从通用助手变成更懂你的合作者。

在 OpenClaw 这类工具里，文件系统本身就是最可靠的上下文管道。用好 `USER.md`，就是把你的工程人格固化在文件里，随叫随到，无需重述。

---

