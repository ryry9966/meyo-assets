---
title: 让 AI 助手更懂你：构建 USER.md 作为 Agent 的上下文基石
feedId: 32751
source: 综合讨论
publishedAt: 2026-08-12
---

## 背景：Agent 很能干，但它不了解你

在使用 OpenClaw、自定义 Agent 或基于 MCP 的自动化流程时，我们常遇到一个尴尬的局面：大模型能力很强，但每次执行任务都需要反复澄清意图，或者给你一堆不符合实际环境的建议。给它一个 shell 执行权限，它可能用 GNU 风格参数，但你的环境只有 BSD 工具；让它写代码，它可能引入一大堆你项目里根本不用的依赖；让它读日志，它可能忽略了你的特殊清洗脚本。

问题不在于模型不够聪明，而在于**缺乏对你和你工作环境的持续认知**。System prompt 虽然可以部分解决，但它通常偏向通用规则，难以承载每个人独特的工作流、偏好和环境变量。于是，我们需要一个更持久、更结构化、更便于维护的“用户手册”。

## 问题：上下文断裂阻碍自动化

设想一个典型场景：你让 Agent 帮你重命名一批文件，然后生成一份报告。它马上动手，但把生成日期的格式弄成了 `YYYY-MM-DD`，而你的团队一直用 `YYYYMMDD`。你纠正它，它改对了这次，下次又忘。

这是因为 Agent 的上下文是会话级别的。每次新对话或任务重启，之前记住的偏好就消失了。即使你通过 MCP 提供了一些资源，那些资源也往往是项目级别的，缺乏用户维度的细节。你需要的是一份**随时可注入的、属于你个人的上下文**，让 Agent 在开始任何操作前，先读懂你。

## 做法：编写并注入 USER.md

USER.md 是一份 Markdown 文件，作用类似于项目根目录的 `README.md` 告诉开发者如何上手，而 `USER.md` 则告诉 AI 助手如何与你协作。它通常包含以下部分：

1. **环境快照**：操作系统、Shell、常用工具版本（如 `zsh 5.9 + GNU coreutils`）、语言运行时等。
2. **风格与约定**：代码命名规范、注释语言、文档格式偏好、日期时间格式等。
3. **项目与工作流**：当前经常操作的仓库、常用命令别名、构建/测试习惯、部署方式。
4. **工具与插件**：你启用的 MCP 服务器、自定义脚本路径、常用 API key 别名等。
5. **约束与边界**：哪些操作需要显式确认、哪些目录/命令永远不要触碰、资源使用限制（如一次最多并行几个 shell）。
6. **个人知识索引**：常用数据源的存储位置、笔记结构、自建短链接规则等。

编写完毕，关键是**如何让 Agent 读到它**。这里有几种工程化注入方式：

- **通过 OpenClaw 的系统级工具注入**：如果你的 Agent 框架支持在每次任务开始加载文件，可以直接指定 `~/.openclaw/user.md` 作为系统上下文的一部分。
- **作为 MCP 资源暴露**：利用 MCP 的 `resources/list` 和 `resources/read` 能力，将 `USER.md` 注册为一个固定的资源 URI（如 `user://profile`），Agent 在需要时主动读取。
- **在 prompt 模板中显式引用**：对于不支持自动加载的框架，可以在任务 prompt 开头加入 `Please read my USER.md at /home/me/.agent/user.md before you act.` 并确保 Agent 有读取本地文件的权限。
- **符号链接至项目**：在项目根目录建立 `.ai/user.md` 的软链，并配合 `.ai/` 目录自动化提交（如 Copilot 的 `.github/copilot-instructions.md`），让团队级 Agent 也能共享部分用户上下文。

推荐的做法是**全局 + 项目分层**。把与特定项目无关的个人偏好放在 `~/.config/openclaw/user.md`，把项目相关的工具链、环境变量、专用规则放在项目中的 `PROJECT_USER.md`，然后在启动 Agent 时按顺序注入，后面的覆盖前面的。

## 踩坑点与经验

1. **信息过载让 Agent 忽略关键内容**  
   一开始容易堆砌太多细节，比如把整个 `alias` 列表都贴进去，结果 Agent 在长上下文中反而丢失重点。**精简为王**，只写那些容易出错的差异化信息。例如，如果你的 `ls` 别名为 `ls --color=auto`，这对 Agent 不重要；但如果你的 `python` 实际指向 `python3.11` 且项目必须用 3.10，这就是必须的。

2. **与 system prompt 冲突**  
   如果 system prompt 强制要求某种输出格式，而你的 `USER.md` 要求另一种，Agent 可能摇摆不定。解决方法是明确优先级：在 `USER.md` 开头声明 `If any instruction here conflicts with a higher-level policy, prioritize my preferences unless security is at risk.` 同时尽量让系统 prompt 的规则更具弹性。

3. **不可执行的静态信息累积**  
   用户的工作环境可能变化，比如升级了 Python 版本、换了默认 Shell。如果 `USER.md` 不更新，就会变成错误的情报。建议将其纳入 dotfiles 管理（如通过 chezmoi 或 yadm），并在日常 dotfiles 更新流程中同步维护。也可以写一个简单的 crontab 脚本，每天检测关键工具版本输出到文件，让 Agent 读取动态生成的部分。

4. **隐私与共享边界**  
   `USER.md` 可能包含 token、路径等敏感信息。如果 Agent 会话本身运行在可信环境，可以直接读取；如果涉及云端 Agent 或团队共享，需要小心脱敏。可以把敏感信息抽离为环境变量，在 `USER.md` 中用占位符引用，例如 `API key: $OPENAI_API_KEY`，Agent 仅知道变量名，实际值由运行时注入。

## 可复用的建议

- **模块化拆解**：别把所有信息塞进一个文件。按功能拆成 `env.md`、`preferences.md`、`tools.md`，再通过一个主索引文件 `USER.md` 用 `# include` 或资源 URI 组合。MCP 资源可以映射多个 URI。
- **为 Agent 提供“帮助命令”**：在 `USER.md` 中定义一个特殊命令，比如 `user_info --show`，Agent 可以通过执行该命令获取最新的环境快照，实现动态更新。
- **加入验证指令**：在 `USER.md` 末尾写一句 `If you have read and understood this file, reply with "User profile loaded."` 这样你立刻知道 Agent 是否真正加载了上下文。
- **版本标记**：在第一行写上 `<!-- u-md version: 20250401 – always update me -->`，方便你检查是否过期。
- **与 OpenClaw 工作流结合**：如果你的 OpenClaw 部署支持 hooks，可以在 session start hook 中注入读取 `USER.md` 的系统消息，并赋予较高权重。

## 总结

给 AI 助手写一份 `USER.md`，本质上是把你个人工程环境的“隐性知识”转化为结构化的、机器可读的上下文。这比每次任务开始前啰嗦一大堆更高效，也比单纯依赖微调或 prompt 工程更可控。它成本极低，一份文件，定期维护，却能让 Agent 的执行准确率显著提升，减少反复纠正的时间。

在 MCP 生态和插件化自动化日趋成熟的今天，让你的 Agent 真正“认识你”，远比让它掌握最新的模型能力更实际。因为强大的工具只有用在正确的上下文里，才是生产力。

---

