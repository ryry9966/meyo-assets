---
title: 给Agent一个可进化的身份：用IDENTITY.md管理OpenClaw的行为与记忆
feedId: 31855
source: 综合讨论
publishedAt: 2026-08-06
---

在构建AI Agent的过程中，最容易被忽视却又最致命的问题是“身份漂移”。你今天让它扮演一个严谨的代码审查员，明天它在多轮对话后可能就变成了一个爱讲冷笑话的闲聊机器人。更麻烦的是，当你把同一个Agent部署到多个环境（开发、测试、生产），或者让多个Agent协同工作时，行为不一致会直接导致任务失败。

过去我们习惯用system prompt来控制Agent。一段自由文本贴在请求的开头，简单直接，但问题也明显：难以版本化、不易追溯变更、无法携带结构化记忆，更不用说在Agent之间复用身份信息了。OpenClaw 提供了一个更工程化的解法：**IDENTITY.md —— 一个声明式的、可进化的Agent身份文件**。

### 不只是 Prompt，而是 Agent 的“源代码”

OpenClaw 的 IDENTITY.md 并非简单地把 system prompt 放到文件里。它是一份混合了 Markdown 与结构化元数据的配置文件，描述一个 Agent 的核心身份、知识边界、行为准则和记忆锚点。它会被框架解析，注入到每次模型调用中，决定了 Agent **如何理解世界、如何做决策、以及如何回忆过去**。

更重要的是，这个文件可以用 Git 管理。你可以像对待代码一样对身份进行分支、合并、打标签，清晰地看到“v1.2 版本的 Agent 增加了对 SQL 查询的审慎校验规则，而 v1.3 版本因为引入 MCP 天气工具而放宽了信息检索时的来源验证”。身份从此有了演进历史，不再是不可控的黑箱。

### 动手：定义一个“运维助手”的 IDENTITY.md

假设我们要构建一个负责 DevOps 任务的 Agent，它需要查询服务器状态、分析日志，并且风格必须严谨，不能虚构数据。以下是一个可工作的 IDENTITY.md 示例：

```markdown
---
name: ops-agent
version: 1.0.0
tools: [mcp-docker, mcp-postgres, mcp-grafana]
environment:
  production: false
  allowed_commands: [docker ps, docker logs, systemctl status]
memory:
  recall: on
  kv_store: sqlite
behavior:
  hallucination: forbid
  style: dry
  max_turns: 20
---

# Core Identity
You are an on-call DevOps assistant. Your sole purpose is to help diagnose infrastructure issues using the provided tools. You never guess. If data is insufficient, you clearly state so and ask for more context.

# Knowledge Boundary
- In-scope: Docker containers, PostgreSQL metrics, Grafana dashboards, systemd services.
- Out-of-scope: Application code, user data, financial queries.

# Safety Rules
1. Never execute destructive commands (rm, drop, kill -9).
2. For any container restart, explicitly confirm the reason and ask for approval.
3. Log all queries and tool invocations to the memory store.

# Memory
- After each session, summarize key findings in a persistent note for future reference.
- When asked "What happened last time?", replay the summary.
```

将这个文件放在项目根目录，OpenClaw 会在初始化 Agent 时读取它。**元数据区**（YAML 前端）定义了工具权限、环境上下文、记忆策略；**正文区**（Markdown）提供自然语言的行为约束。这种结构使得身份既可以被程序解析，也可以被人直接阅读和修改。

### 踩坑记录

在多个实际项目中反复使用 IDENTITY.md，有几点值得注意：

1. **YAML 缩进是硬伤**  
   元数据区域使用 YAML 语法，很多首次接触的开发者会在 `allowed_commands` 这类列表项上缩进错误，导致文件被静默忽略。一个检查技巧：用 `yq eval . identity.md` 快速验证 YAML 部分是否合法。

2. **`memory` 配置若隐若现**  
   开启了 `memory: recall: on` 但忘记配置 `kv_store`（或指向无效路径），Agent 会在每次调用时尝试写入记忆而失败，却不会明显报错。结果就是记忆永远为空，但表面一切正常。务必定时检查记忆库文件是否增长。

3. **正文与元数据的冲突**  
   正文里写 “You must be friendly”，但元数据里 `style: dry`，模型会困惑。OpenClaw 的解析顺序是**正文覆盖元数据**（取决于具体版本），若未来改变，行为可能突变。建议只在一处定义同类型的约束，避免二义性。

4. **环境变量注入失效**  
   `environment` 下的字段如果不加引号，被当成布尔或数字，导致传入空值。例如 `staging: true` 没问题，但 `server_list: web01,web02` 会被当作字符串，可能不符合预期。值得用 `!env` 标签显式标记或全部加引号。

### 可复用的工程化建议

- **模板化与继承**：为不同场景维护基础身份模板（base-devops, base-sre），具体 Agent 的 IDENTITY.md 通过 include 或引用覆盖特定字段。OpenClaw 支持 `extends` 指令，可以创建轻量派生身份。
- **与 MCP 工具解耦**：元数据中的 `tools` 列表只做权限声明，实际工具注册在 config.yaml 中管理。这样做可以防止身份文件膨胀，也便于在不同环境启用不同工具集而不必改动身份。
- **记忆的自动化复盘**：编写一个 Cron 脚本，每小时拉取 Agent 的记忆摘要，推送到团队频道。这不仅能跟踪身份演化效果，还能提前发现 Agent 的行为异常。
- **版本发布 Checklist**：更新 IDENTITY.md 后，在 CI 流程中加入三步验证：YAML 合法性检查、与预设规则对齐（不允许 `hallucination: allow` 等危险配置）、在隔离环境中运行 5 个典型任务查看行为一致性。

### 总结

给 AI 一个可进化的身份，本质上是将 Agent 的“性格”从代码中解耦出来，变成一份可追溯、可测试、可协作的资产。OpenClaw 的 IDENTITY.md 让这件事变得低门槛且工程化，但真正的收益取决于你是否像对待生产代码一样对待它：**版本控制、自动化测试、渐进式演进**。

当你的 Agent 开始因为身份清晰而减少犯错，当团队可以复用同一份身份在不同项目中得到一致表现，你才真正拥有了一个“可进化的 AI 成员”，而不是一个每次都需要重新教它做事的对话工具。

---

