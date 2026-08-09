---
title: 让 Agent 真正了解你：为 OpenClaw 搭建 USER.md 个人上下文系统
feedId: 32268
source: 综合讨论
publishedAt: 2026-08-09
---

## 背景：你的 Agent 其实不认识你

不管你用的是 OpenClaw、MCP 工具链还是自建的 Agent 编排，所有 AI 助手都面临同一个问题：**每次新对话都是一张白纸**。即使你在上一次对话中花了十分钟解释自己的工作栈、偏好和项目结构，下个 session 一切归零。反复自我介绍不只消耗耐心，还会让自动化脚本、工具调用失去连续性——Agent 不知道你常用的 shell 别名、默认的数据库连接方式，甚至记不住你讨厌 `yarn` 而选择 `pnpm`。

为了解决这个问题，社区里常见做法是维护一个“系统提示词”模版，但它的缺点很明显：要么臃肿到挤占上下文，要么分散在不同项目里难以统一复用。我们需要一种**轻量、可版本化、能被 Agent 自动加载的“用户身份文件”**。我把它称为 `USER.md`。

## 问题拆解

一个面向 Agent 的用户文件，至少要解决四个痛点：

1. **首轮冷启动**：Agent 在对话开头就能获取你的基础信息，无需提问。
2. **工具链一致性**：自动提示你习惯使用的命令、环境变量前缀、本地服务端口等。
3. **项目感知**：在不同仓库下切换时，能结合目录内配置给出智能建议。
4. **安全性**：不把密钥、令牌直接写进容易被忘掉删除的纯文本。

## 做法：三步搭建可被 Agent 读取的 USER.md

### 1. 确定文件位置与加载规则

推荐将 `USER.md` 放在用户主目录下，路径如 `~/.openclaw/USER.md`，这样所有基于该身份启动的 Agent 实例都能统一读取。如果你的 OpenClaw 支持 `workspace` 或 `project` 级配置，可以在项目根目录放置 `AGENT.md` 作为补充，但 `USER.md` 负责通用人设。

加载方式取决于你使用的 Agent 框架。以 OpenClaw 为例，可以在 `agent.yaml` 中通过 `pre_prompt_file` 或 `system_message_append` 关键字将文件内容注入。如果你在使用 MCP 服务端，可以让 `context-injection` 插件在每次 `generate` 请求前读取该文件并拼接到消息列表中。

简单的 shell 启动脚本示例：

```bash
export OPENCLAW_USER_CONTEXT="$(cat ~/.openclaw/USER.md 2>/dev/null)"
```

然后在 agent 配置中引用该环境变量。

### 2. 内容结构设计

一份工程化的 `USER.md` 不是自传，而是**给模型看的结构化上下文**。信息密度要高，避免散文。建议采用 YAML front matter + Markdown 段落：

```markdown
---
name: "Alex"
pronouns: "they/them"
timezone: "Asia/Shanghai"
default_shell: "zsh"
editor: "nvim"
preferred_tools:
  - pnpm
  - tgpt
  - fzf
  - gh
  - jq
avoid_tools:
  - yarn
  - npm
project_roots:
  - ~/workspace/oss
  - ~/workspace/internal-tools
---

# 工作偏好
- 所有代码示例优先使用 TypeScript + ESM。
- 命令示例用 `bash` 块，不要 `sh`。
- 解释概念时先给一句话总结，再展开。
- 我不会要求你“像专家一样”，把方案讲清楚就行。

# 常用别名与环境
- `dc` 指 `docker compose -f ~/infra/compose.yml`
- `tb` 是 `task build` 的别名
- 本地 PostgreSQL 在 `localhost:5433`，用户 `dev`
- 所有 API 测试用 `httpie`，不要用 `curl`

# 项目速查
- `~/workspace/oss/openclaw`：主仓库，Go 项目，使用 `Makefile`
- `~/workspace/internal-tools/api-server`：Node 服务，Fastify，入口 `src/index.ts`
```

要点：**用模型理解的方式编码**，比如“不要用 curl”比“我不喜欢 curl”更直接。

### 3. 与 MCP / 插件联动

有了文件只是静态信息，真正让它“活”起来需要与工具联动。你可以写一个轻量 MCP 服务，暴露 `user_context` 资源，把 `USER.md` 的内容作为资源读取，Agent 可以通过 MCP 协议随时获取最新版本。这样做的好处是，即使上下文里没有注入，Agent 也能在需要时主动拉取。

如果你在用 OpenClaw 的插件系统，可以写一个 `user-profile` 插件，在每次会话开始时执行一次 `read` 调用，并把内容缓存到会话的元数据中。注意不要每次都重读文件，避免 I/O 延迟累积。

## 踩坑点

1. **上下文溢出**  
   `USER.md` 越长，留给真正工作的窗口越小。控制文件在 500–800 字以内，超过的部分可以拆分为 `PROJECT.md` 放在具体项目下。Agent 只需要知道“去哪里找详细配置”，而不是记住所有细节。

2. **敏感信息泄漏**  
   绝对不要把 `OPENAI_API_KEY`、数据库密码写进文件。如果你的 Agent 需要用到某个 token，可以用占位符 `env:MY_TOKEN`，并在启动脚本里通过环境变量注入，由 Agent 运行时解析。这样文件可以安全地放到 dotfiles 仓库中版本管理。

3. **路径与机器绑定**  
   `~/` 开头的路径在不同机器上可能不一致。尽量使用环境变量或 `${HOME}` 占位，在加载时做一次替换。如果 Agent 运行在远程容器里，`/home/alex` 和本机路径不同会更头疼，建议只使用相对项目根或容器内路径。

4. **过期信息**  
   你昨天还在用 `vim`，今天可能切到 `helix`，但 `USER.md` 忘记更新。Agent 会持续给出错误建议。解决方法是养成每次变更工具时更新文件的习惯，或者写个 hook：`.zshrc` 里检测常见编辑器变化后提示更新。

## 可复用建议

- **模板化**：把你的 `USER.md` 拆为头部（静态人设）、中部（工具链）、尾部（项目索引）。团队内不同成员只需修改头部，其他部分可以共享并从团队骨架仓库继承。
- **用脚本自动搜集**：比如通过 `hostname`、`whoami`、`which` 等命令在每次登录时自动生成一段“环境摘要”写入临时文件，Agent 可以读取该摘要而不是手工维护的 `USER.md`。适合懒人和多变的环境。
- **将 `USER.md` 视为接口规范**：不是给人类看的笔记，而是 Agent 的配置项。改变某个偏好时，想想“Agent 是否能正确解析这个变更”。保持键名稳定，如 `preferred_tools` 不要忽然改成 `fav_cli`。
- **与团队共享基础配置**：如果你的团队都用 OpenClaw，可以维护一个 `base-user.md`，大家 fork 后个性化。这样基础的代码风格、部署命令等保持一致，减少 Agent 的混淆。

## 总结

`USER.md` 不是系统提示词的替代品，而是 Agent 的“驾照”——一份轻量、结构化、机器可读的身份文件。它让你的 AI 助手不再从零开始理解你，而是带着上下文进入每一次交互。在实践中，合理的结构、谨慎的安全边界和与 MCP/插件的联动，能让它从静态文本进化成可查询的活接口。对于追求效率的工程师来说，花半小时写一份 `USER.md`，可能比连续两周每天向助手解释“对，我还在用 pnpm 而不是 yarn”划算得多。

你的 Agent 值得认识真正的你。

---

