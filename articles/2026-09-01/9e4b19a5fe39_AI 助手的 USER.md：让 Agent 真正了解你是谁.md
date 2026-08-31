---
title: AI 助手的 USER.md：让 Agent 真正了解你是谁
feedId: 35615
source: 综合讨论
publishedAt: 2026-09-01
---

## 背景：Agent 不认识你

OpenClaw 这类 Agent 框架的默认行为是“通用助手”：它能调 MCP 工具、跑插件、读写文件，却不了解你的系统路径、常用命令、代码风格和操作边界。于是每次新会话都要重新解释环境，Agent 还可能在错误的目录执行命令，或给出不符合你习惯的方案。

USER.md 的作用就是把“关于你”的静态上下文沉淀成文件，让 Agent 在启动或需要时读取。它和项目级 AGENTS.md 的区别在于：USER.md 描述个人偏好与机器环境，AGENTS.md 描述项目约定。

## 问题：上下文靠对话口述，难以复用

没有 USER.md 时，常见三类问题：

1. **重复说明**：每次都要告诉 Agent 用 zsh 而非 bash、项目在 ~/dev 下、Python 用 uv 管理。
2. **环境误判**：Agent 默认使用系统 Python 而不是 venv，或在错误目录创建文件。
3. **边界不清**：Agent 可能主动修改 dotfiles、执行危险命令，因为你没显式声明“不要动哪些目录”。

这些问题不是模型能力问题，而是缺少一份稳定的用户说明书。

## 做法：创建并接入 USER.md

### 1. 确定存放位置

推荐放在用户级配置目录，例如 `~/.openclaw/USER.md`。如果 OpenClaw 支持工作区配置，也可以在项目内放 `.openclaw/USER.md`，但个人偏好更适合全局。确保 Agent 有权限读取该文件。

### 2. 写一份最小可用的 USER.md

内容不必长，建议控制在 200 行以内。结构可以是：

```markdown
# User Profile

## Identity
- role: backend developer
- timezone: Asia/Shanghai
- os: macOS / Ubuntu

## Environment
- shell: zsh
- editor: neovim
- code_dir: ~/dev
- package_manager: uv / pnpm

## Preferences
- commit style: conventional commits
- language: zh-CN for docs, English for code
- before running scripts: show diff first

## Boundaries
- never modify: ~/.ssh, ~/.gnupg, /etc
- never run: rm -rf, force push to main
- ask before installing system packages

## Common Commands
- activate venv: source .venv/bin/activate
- run tests: pytest -q
- build: make build
```

这份文件是给 Agent 看的，不是给人看的，所以用短句、列表、明确动词。

### 3. 让 Agent 读取

可以在 OpenClaw 的 system prompt 或初始化指令中加一句：

```text
Before any task, read ~/.openclaw/USER.md if it exists. Follow the environment, preferences and boundaries described there.
```

如果 OpenClaw 使用 MCP 文件系统工具，确认该路径在允许读取范围内。也可以将 USER.md 内容直接注入 system prompt，但文件方式更容易维护。

### 4. 随环境变化更新

换机器、改 shell、换包管理器后，及时更新 USER.md。最好把它纳入 dotfiles 仓库，和 .zshrc、.gitconfig 一起版本管理。

## 踩坑点

1. **文件过长**：几百行的 USER.md 会消耗大量上下文窗口，Agent 也可能抓不住重点。保持精炼，只写影响决策的信息。
2. **过时内容**：写“默认使用 npm”，后来改用 pnpm，Agent 仍按旧信息执行。建议每季度 review 一次。
3. **敏感信息泄露**：不要把 API key、密码、私钥路径明文写进 USER.md。使用环境变量或密钥管理工具，文件里只写引用名。
4. **与项目配置冲突**：如果项目 AGENTS.md 说用 yarn，USER.md 说用 pnpm，Agent 可能困惑。可以在 USER.md 中声明优先级：项目规则优先于个人偏好。
5. **路径硬编码**：不同机器路径不同，例如 macOS 和 Linux 的 home 路径。可以用 `~` 或环境变量，而不是绝对路径。

## 可复用建议

- **分层管理**：全局 USER.md 放个人偏好，项目级 AGENTS.md 放项目约定，会话级指令放具体任务要求。
- **使用模板**：为团队提供统一模板，降低编写成本。
- **配合 MCP 动态获取**：静态 USER.md 无法覆盖所有信息，例如当前分支、服务端口。可以用 MCP 工具动态读取这些状态，让 Agent 不依赖过时记录。
- **限制作用域**：在 USER.md 中明确“默认不要修改哪些路径”，比事后再补救更有效。

## 总结

USER.md 是低成本、高收益的 Agent 基础设施。它把“你希望 Agent 了解的信息”从对话中抽离出来，形成可版本化、可复用的静态上下文。对于 OpenClaw 用户来说，花半小时写一份 USER.md，能明显减少后续每次会话的纠错成本。关键是保持精炼、及时更新、不存敏感信息，并与项目级配置做好分工。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/d4980fa43b6b0904.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/a60eb8a49411fcc1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/e25f03cc732a31c8.png)

