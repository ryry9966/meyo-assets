---
title: OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册
feedId: 32371
source: 综合讨论
publishedAt: 2026-08-10
---

# OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册

## 为什么你的 Agent 总在跑偏

在用 OpenClaw 把日常开发、文档、运维流程逐步交给 Agent 之后，一个反复出现的问题是：**Agent 缺少对当前工作空间的稳定认知**。每次对话都要重新交代“这个项目用什么构建系统”、“单元测试怎么跑”、“日志文件在哪里”、“MCP 工具怎么用”，不仅浪费 token，还容易出现前后不一致的输出——今天按 Google Java Style 生成代码，明天又切回默认风格，后天连工具名都搞错。

根本原因不是模型能力不够，而是我们没有像对待新加入团队的开发者那样，给 Agent 准备一份“新人指引”。OpenClaw 的 AGENTS.md 就是用来解决这个问题的：它是一份放在仓库里的、专门写给 Agent 的工作空间使用手册，Agent 在每次会话中自动加载，作为理解项目上下文和行为约束的稳定锚点。

## AGENTS.md 能做什么

把它类比成 `.editorconfig` + `CONTRIBUTING.md` + `Makefile` 的合体，但读者是 AI。在 OpenClaw 中，AGENTS.md 不只是静态文件，它可以被 Agent 逐段解析，影响工具选择、文件操作、代码风格乃至 MCP 服务端行为。一份合格的 AGENTS.md 通常至少要覆盖这些维度：

- **项目与技术栈**：这是什么项目，主要语言/框架/运行时版本，外部依赖概览。
- **目录结构约定**：代码在哪里、测试在哪里、配置文件在哪里、文档在哪里，代理操作文件时的默认根目录。
- **编码规范与风格**：引用的 linter/formatter 配置，命名约定，注释要求，禁止项。
- **常用命令**：构建、测试、lint、启动本地服务、数据库迁移等命令及执行环境说明。
- **MCP 工具使用指引**：启用了哪些 MCP server，各自的能力边界，调用时的参数习惯、常见错误处理。
- **任务模式与示例**：Agent 执行高频任务（如修复一个测试、新增 API 端点、写 changelog）时的推荐流程。
- **约束与红线**：绝对不能动的文件、禁止的 shell 命令、敏感信息处理策略。

重点在于“精确”而不是“说教”。每一条都应是可以被 Agent 直接执行或检查的，而不是“你要认真负责”这种模糊表述。

## 怎么落进你的 OpenClaw 工作流

**1. 放置位置与加载方式**  
一般放在项目根目录的 `AGENTS.md`，OpenClaw 的配置里通过 `agent.workspace_handbook` 或类似字段指定路径。多仓库场景下，每个仓库一份，Agent 进入对应工作空间自动加载。也可以用 `.openclaw/` 目录存放，配合 `include` 语法拆分成多个文件。

**2. 从最小可用的版本开始**  
不要一上来就写成百科全书。初始版本只写让 Agent 做对一件小事的必要信息，比如“如何在这个仓库里启动单元测试”。然后让 Agent 实际执行这件事，观察它是否遵循了 AGENTS.md 的描述。如果它忽略了某条，就把那条写得更严格或更靠前。

**3. 用模板降低维护成本**  
可以准备一个 AGENTS.md 骨架，每次开新项目时复制并填空。下面是一个精简示例结构：

```markdown
# Project: my-service
- Language: Go 1.22
- Build: `go build ./cmd/server`
- Test: `go test ./...` (no race flag by default)
- Lint: `golangci-lint run`
- Directories:
  - internal/handler: HTTP handlers
  - internal/service: business logic
  - internal/store: data access
- MCP Tools:
  - file-system: allowed, but never write outside the repo root
  - database: postgres-mcp, connection string from `.env`
- Code Style:
  - exported functions must have a comment
  - errors are wrapped with `fmt.Errorf` context
- Red Lines:
  - never delete `.env` or `*.pem` files
  - never commit secrets
```

**4. 纳入版本控制与迭代**  
AGENTS.md 是代码的一部分。每次项目约定或工具链变更时同步更新它，并且在 PR 里说明改动对 Agent 行为的影响。定期抽查：故意让 Agent 做一件违反 AGENTS.md 的事，看它会不会主动拒绝或指出问题。

## 踩过的坑

- **大而全的文档反而被忽略**：Agent 的上下文窗口有限，过长或结构混乱的 AGENTS.md 会被模型在注意力分配中“遗忘”。实测中，超过 3KB 的文本如果没有清晰的段落和列表，Agent 遵守率显著下降。拆分为多个文件并在主文件里用 `@include` 引用更有效。
- **相对路径陷阱**：如果 AGENTS.md 里写了 `cd scripts && ./deploy.sh`，但 Agent 的工作目录不是项目根，就会执行失败。所有路径应尽量用相对于 repo 根目录的形式，或通过环境变量约定。
- **指令冲突**：同时存在 AGENTS.md 和系统 prompt 中的全局规则（如“生成代码后自动运行测试”），可能导致 Agent 行为摇摆。需要明确优先级，例如在工作空间手册中写明“本文件中的规则覆盖全局默认设定”。
- **敏感信息泄露风险**：有人在 AGENTS.md 里写了数据库连接示例，直接带真实凭据。务必用占位符或环境变量，并把 AGENTS.md 视为公开文档对待。
- **多 Agent 协作不兼容**：如果一个工作空间里既有编码 Agent 又有评审 Agent，通用 AGENTS.md 可能不够。可以为不同 Agent 用条件段落标记，或利用 OpenClaw 的 profile 按角色加载不同片段。

## 可复用的工程化建议

1. **分层设计**：基础层（通用规则） + 模块层（子目录规则）组合，Agent 进入子目录时叠加加载。
2. **版本标记**：在文件头部加 `<!-- version: 2.1 -->`，方便追踪 Agent 行为变更的原因。
3. **可测试的指令**：每一条“行为约束”都应有办法让人或脚本验证 Agent 是否遵从。例如“生成的 JSON 必须包含 `version` 字段”，就可以通过解析 Agent 输出自动检查。
4. **与 MCP 能力对齐**：AGENTS.md 里描述的 MCP 工具必须与 `mcp.json` 中实际启用的 server 一致，否则 Agent 会幻想出不存在的工具。
5. **定期淘汰**：每季度 review 一次，删除不再使用的命令、旧依赖说明，防止信息腐化。

## 总结

AGENTS.md 不是一份“一次写完再也不用看”的文档，而是 Agent 时代代码库的交互界面定义。它的目标很明确：让 Agent 在进入工作空间时，像熟手一样理解规矩，而不是一个需要不断纠错的实习生。投入精力打磨这份手册，换回的是 Agent 输出的一致性、可控性，以及在多代理协作时显著降低的沟通成本。如果你的 OpenClaw 工作空间还没有 AGENTS.md，现在就是最好的创建时机——从一个最小的、能让 Agent 跑通单元测试的手册开始。

---

