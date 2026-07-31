---
title: AI 助手的 USER.md：让 Agent 真正了解你是谁
feedId: 31084
source: 综合讨论
publishedAt: 2026-07-31
---

# AI 助手的 USER.md：让 Agent 真正了解你是谁

## 背景

在 OpenClaw 等 Agent 自动化实践里，我们花大量精力给 Agent 接 MCP 工具、配插件、写 Workflow，但常常忽略一个基础问题：**Agent 并不了解操作它的人**。每次新对话都是一张白纸，你需要反复解释自己的技术栈、目录习惯、命名偏好，甚至操作系统类型。这就像每次都跟一个新同事重新介绍自己，效率很低。

常见的应对方式是往 system prompt 里塞个人信息，但 system prompt 容易膨胀、难以复用，也不利于在不同 Agent 实例间共享。于是，借鉴 Unix 的 `~/.bashrc`、Git 的 `.gitconfig` 思路，**USER.md** —— 一份用户自述文件，逐渐成为我工程化个人 Agent 上下文的核心组件。

## 问题：Agent 真的不懂你

举个例子：你让 Agent 写一个启动开发环境的脚本。它不知道你用 zsh 还是 fish，默认给你一个 bash 脚本，你只好手动改成 zsh，并说明“以后默认用 zsh”。下一次新对话，同样的问题再次出现。再如，你想让它生成一份周报，它给出英文、技术细节繁多的版本，而你其实需要中文、要点式的摘要。反复调整不仅耗时，也消磨了对 Agent 的信任感。

这些问题的根源不是 Agent 能力不足，而是它缺少关于**你**的上下文。工具可以靠 MCP 标准化，数据可以靠 RAG 检索，但“你是谁”这个元信息，需要一种低摩擦、可跨会话持久化的机制来承载。

## 做法：构建并注入 USER.md

核心思路很简单：写一个 Markdown 文件描述你自身，然后通过 MCP 的资源（Resources）或工具（Tools）机制，让 Agent 在每次对话启动时加载。下面是一个经过几次迭代的实践步骤。

### 1. 编写 USER.md

放在 `~/.openclaw/user.md` 或项目根目录的 `.claw/user.md`。内容建议包含：

- **身份与角色**：称呼、职位、常用时区、首选语言（zh-CN 等）
- **环境信息**：操作系统（macOS/Linux）、Shell（zsh）、默认编辑器（neovim）、包管理器（brew/pnpm）
- **技术偏好**：常用语言/框架、代码风格（比如总是用 async/await）、是否要求错误处理说明、注释语言
- **输出习惯**：文档详略程度、是否解释每一步、报告格式、回复语言（中文/English）
- **当前项目目标**：简要描述，避免跑偏
- **约束与边界**：禁止自动执行的命令、不能访问的目录、不要使用的工具

示例结构：

````markdown
---
name: "OpenClaw 贡献者"
os: "macOS 14"
shell: "zsh"
editor: "neovim"
language: "zh-CN"
---
# 我是谁
- 全栈开发者，主要使用 TypeScript、Go
- 偏好函数式风格，错误必须显式处理
- 代码注释用中文，日志用英文
- 项目根目录在 ~/workspace/openclaw

# 规则
- 所有脚本默认使用 zsh，不要用 bash
- 生成文档时优先给出要点，不要长篇大论
- 涉及文件操作前先询问，除非我明确要求直接执行
````

使用 YAML front matter 便于工具解析，也可以纯 Markdown，取决于集成方式。

### 2. 通过 MCP 提供给 Agent

关键在于让 Agent 可以**主动获取**而不是硬编码到 system prompt。最简方式是用一个文件读取 MCP Server，将 `user.md` 暴露为 resource，URI 如 `user://profile`。在 OpenClaw 的配置里加入：

```yaml
mcp_servers:
  - name: user-profile
    command: npx
    args: ["-y", "@anthropic/mcp-server-filesystem", "/Users/you/.openclaw"]
```

然后在 Agent 的指令中加入：“启动时通过 `user://profile` 资源读取用户偏好，并严格遵循其中的约束。” 如果 MCP Server 只提供工具，可以用 `read_file` 工具主动读取。另一种模式是在对话开始后，让 Agent 自动调用 `get_user_profile` 工具填充上下文。

### 3. 验证 Agent 行为

重启 Agent，问一个偏好相关问题：“帮我写一段检查端口的脚本。” 如果 Agent 输出 zsh 脚本并附中文注释，说明 USER.md 生效了。再试一个边界：“执行 `rm -rf /tmp/test`” 看它是否先询问。可以根据实际情况微调 USER.md 的粒度。

## 踩坑与取舍

实践中踩过几个坑，值得提前规避：

- **Token 浪费**：把整个技术百科塞进 USER.md，每次对话平白消耗上千 tokens。保持精简，只放真正影响决策的信息，动态数据（如当前 git 分支）交给专门的工具。
- **隐私泄露**：绝对不要在 USER.md 存放密钥、Token 或私人服务器地址。即使文件在本地，MCP Server 可能将其内容发送至模型 API，风险不可控。敏感信息用专门的 secret 管理工具，并限制 Agent 访问。
- **多层级覆盖冲突**：如果同时存在全局 `~/.openclaw/user.md` 和项目级 `.claw/user.md`，Agent 可能混淆。建议在 system prompt 中明确优先级：项目级覆盖全局，或者要求 Agent 合并时以项目级为准。
- **格式解析错误**：有的模型对 YAML front matter 解析不稳定，有时会忽略。稳妥的做法是仍然在 Markdown 正文中用清晰的小标题列出关键点，YAML 作为辅助。
- **更新滞后**：文件修改后，如果 MCP Server 缓存了内容，Agent 可能读到旧版本。简单场景下，重启 MCP Server 即可。更自动化的做法是使用支持文件监听的 MCP Server，或在每次对话开始时强制重新读取。

## 可复用建议

想让 USER.md 在团队或社区中推广，可以考虑：

1. **提供分类模板**：developer.md、designer.md、student.md，覆盖常见角色，降低编写门槛。
2. **项目级与用户级组合**：个人配置放全局，项目特殊约束放 `.claw/` 下，Agent 合并两者，兼顾复用与针对性。
3. **绑定 MCP Resources**：在 OpenClaw 插件生态中封装一个 `user-profile` 插件，自动读取标准路径的 USER.md 并暴露 resource，一键安装即可用。
4. **动态扩展**：除了静态文件，可以结合 Calendar MCP 将当前状态注入，但始终以用户显式声明的偏好为准，避免 AI 猜测。

## 总结

USER.md 是一种低成本、高收益的 Agent 个性化方案。它把“我是谁”从 prompt 工程的一次性胶水代码，变成了可维护、可共享、可跨会话复用的配置文件。在 OpenClaw 这种以 MCP 为核心的 Agent 框架里，通过资源机制加载 USER.md，既保持了架构的解耦，又让 Agent 真正朝着“你的副驾驶”方向迈进了一步。如果你已经在用 MCP 给 Agent 接各种外部能力，不妨花一刻钟写下自己的 USER.md，很可能下一次对话你就会感到明显不同。

---

