---
title: OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册
feedId: 31953
source: 综合讨论
publishedAt: 2026-08-07
---

## 背景

让 AI 助手真正“理解”一个工程项目的上下文，始终是 Agent 落地中最容易卡住的一环。无论你用 OpenClaw 编排多步 Agent 任务，还是通过 MCP 让模型操作文件系统、数据库或内部 API，都会遇到同样的问题：每次新会话都要重新解释一遍项目结构、技术栈、规范约束，稍有不慎，模型就会给出看似合理但完全不符合项目要求的输出。

OpenClaw 给出的解法之一，是把这些上下文沉淀到一个工程文件里——**AGENTS.md**。它不是给人类看的 README，而是写给 AI 的工作空间使用手册。运行时，Agent 会在会话启动阶段自动注入这份文件的内容，作为全局记忆，贯穿后续所有工具调用和代码生成。

## 问题

没有 AGENTS.md 的工作空间，AI 助手通常会遇到以下典型问题：

- **工具盲**：环境中挂载了多个 MCP server（例如文件系统、Postgres、Redis），但模型不知道哪些可用、各自能力边界在哪，要么不敢用，要么乱用。
- **规范游离**：项目明明强制使用 `pnpm`、`vitest`、`TypeScript strict` 模式，模型却老是生成 `npm` 命令、`jest` 用例或 `any` 泛型。
- **目录误操作**：不知道 `/generated` 目录禁止手动修改，或者 `/migrations` 下的文件必须保持时间戳顺序。
- **安全红线缺失**：不清楚哪些环境变量可以暴露给日志，哪些数据需要脱敏，很容易在生产排查时直接输出明文密钥。

这些都不是模型能力问题，而是**上下文缺口**。反复口头纠正不仅浪费 token，更破坏了 agent 调用的可靠性。

## 做法 / 步骤

### 1. 文件位置与发现机制

OpenClaw 支持项目根目录下的 `AGENTS.md`，也兼容放在 `.openclaw/AGENTS.md`。建议放在根目录，这样即使不通过 OpenClaw 入口，其他兼容该规范的 agent 框架也能复用。

文件采用纯 Markdown，无自定义 frontmatter，Agent 加载时会把全文注入 system prompt 的尾部，优先级高于模型内置指令。

### 2. 模板结构

一份可用的 AGENTS.md 无需追求大而全，但要覆盖模型容易出错的几个维度。推荐以下最小结构：

```markdown
# Project: my-service

## Identity & Goals
- 面向内部运维的 Node.js 微服务，提供告警聚合与工单派发能力
- 所有对外接口必须幂等，返回格式统一为 `{ code, data, message }`

## Environment
- Node.js 20 LTS，包管理工具 pnpm
- 本地开发使用 `docker-compose -f dev-infra.yml up -d`
- 必需环境变量见 `.env.example`，请勿在日志/回显中输出 DATABASE_URL 和 SECRET_KEY

## Directory Constraints
- `src/generated/` 由 OpenAPI 工具自动生成，**严禁手动修改**
- `migrations/` 中的文件必须按 `YYYYMMDDHHmmss_description.sql` 命名
- 所有测试文件放在 `__tests__/` 目录下，与源文件同名加 `.test.ts` 后缀

## Available Tools (MCP)
- `filesystem`: 可读写 `src/` 和 `tests/`，不允许操作 `/etc` 或根目录
- `postgres`: 只读权限，查询模板需显式使用参数化，禁止拼接 sql
- `slack`: 仅用于发送通知，不允许读取历史消息

## Code Conventions
- TypeScript strict mode，禁止 `any`，函数必须显式声明返回类型
- 错误处理：统一抛出自定义 `AppError`，控制器层不得直接捕获 `Error`
- 命名：文件名 kebab-case，变量 camelCase，React 组件 PascalCase

## Testing
- 运行测试：`pnpm test -- --coverage`，要求行覆盖率 > 80%
- 新增模块必须有对应单元测试，不写 snapshot 测试
- 涉及数据库的测试使用 `testcontainers` 拉起的真实 Postgres 实例

## Common Scenarios
- 新增接口：在 `src/routes/` 下创建路由文件，在 `src/controllers/` 编写控制器，使用 `ajv` 校验入参，响应用 `res.ok(data)` 包裹
- 添加数据库迁移：执行 `pnpm migrate:create <description>`，生成迁移文件后填入 SQL，再运行 `pnpm migrate:up`
```

### 3. 在 OpenClaw 中生效

OpenClaw 的默认配置会扫描项目根目录的 `AGENTS.md`。如果你使用了自定义的 workspace 规则，确认配置中包含了：

```yaml
agent:
  context_files:
    - path: AGENTS.md
      inject: always
```

配置完成后，可以在 OpenClaw CLI 下用简单任务验证，例如：“请按项目规范在 src/routes 下新增一个健康检查接口”。观察模型是否自动使用了 `res.ok()` 包裹响应、是否遵循了 TypeScript strict 风格。

## 踩坑点

- **把 AGENTS.md 写成了“说明书”而非“约束”**  
  大量自然语言描述项目业务逻辑，但缺少可被模型直接引用的规则（如“禁止”、“必须”、“统一使用”）。AI 更依赖格式明确的指令，而非散文式背景介绍。

- **和 README 边界模糊**  
  AGENTS.md 不应重复 README 已经承载的人类阅读信息。它的核心读者是 AI，强调可执行的操作约束，而不是功能介绍。典型反例：“我们使用了高性能的缓存层”应改为“本地缓存统一使用 `lru-cache`，TTL 默认 5 分钟，key 前缀为 `cache:`”。

- **环境/路径硬编码**  
  直接写死了 `http://localhost:5432` 等本地地址，但 agent 实际运行在容器或远程开发环境中，导致 MCP 连接失败。应使用环境变量占位符，如 `${POSTGRES_URI}`，并注明从哪里读取。

- **更新滞后**  
  项目引入了新的 MCP tool 或改动了构建命令，但 AGENTS.md 还停留在旧版本，agent 会持续尝试已经废弃的接口。建议在 CI 里加一个 lint 步骤，检测 AGENTS.md 中引用的命令是否仍然可用。

## 可复用建议

1. **用 Markdown 注释隔离 AI 专属内容**  
   如果部分指令只希望 AI 看到，可以放在 `<!-- AI: xxx -->` 注释里，OpenClaw 解析时会保留。

2. **把示例代码直接嵌在规则里**  
   比如在“禁止直接操作 DOM”后附上正确和错误的对比代码块，模型对正样本和负样本的遵循度会显著提高。

3. **控制文件长度**  
   超过 500 行的 AGENTS.md 会被模型稀释注意力，建议拆分为多个模块文件，再用 `include` 类指令引入（OpenClaw 支持 `<!-- include: .openclaw/rules/testing.md -->` 语法）。

4. **纳入版本控制与 Code Review**  
   AGENTS.md 的变更应走 PR 评审，因为它直接影响 agent 的行为和输出质量。每次新增 MCP 工具或改动项目强制规范，都必须同步更新该文件。

5. **定期做“Agent Walkthrough”**  
   每隔一个迭代，让 agent 按 AGENTS.md 从头执行一次典型任务（如新增接口 + 测试），检查是否仍然奏效，同时发现隐藏的环境变化。

## 总结

AGENTS.md 本质上是一种**低摩擦的工程化上下文注入**。它不依赖某个特定模型的微调，也不要求开发者学习新的 DSL，只是把原本散落在团队口头约定、Slack 频道、Code Review 评论里的约束，翻译成一份 AI 能稳定消费的结构化文档。

在 OpenClaw 生态里，这份文件让 Agent 从“需要全程手把手指导”变成“可以独立完成符合项目标准的子任务”。对于 MCP 工具集较大的团队，效果尤其明显：模型知道什么时候该调数据库，什么时候该退回到文件系统，什么时候应该拒绝执行。

如果你已经开始用 OpenClaw 编排 MCP 或多步 agent 流程，但总觉得“模型很聪明却不听话”，不妨先花半小时整理一份 AGENTS.md——相比反复在 prompt 里纠正，这才是真正的一次投入、持续收益。

---

