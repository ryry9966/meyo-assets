---
title: 用 AGENTS.md 给 OpenClaw 上锁：让 AI 代理读懂你的项目意图
feedId: 32232
source: 综合讨论
publishedAt: 2026-08-09
---

在与 OpenClaw 这类 AI 代理协作一段时间后，你大概率会遇到这样的场景：明明口头描述了清晰的意图，代理却把临时脚本塞进 `src/`，或者擅自`npm install`了一个你从未打算引入的包。更糟糕的是，它在一次“小改动”里顺手把关键测试文件删了，还理直气壮地回复 “Done”。

问题不在于模型不够聪明，而在于它缺少一份**工作空间的静态操作手册**。于是，我们引入了 `AGENTS.md`——一个专为 OpenClaw 代理设计的项目级导引文件。

---

## 为什么需要 AGENTS.md

在无约束环境下，OpenClaw 代理的行为依赖两个来源：系统提示（system prompt）和对话上下文。系统提示由运行环境提供，对所有项目通用；对话上下文则很容易被长任务稀释，导致代理在执行后期忘记最初的约束。`AGENTS.md` 恰好填补了“项目专属、始终生效、结构化”的空白。

它像 `README.md` 之于开发者，但是写给 AI 看的。作用有三：

1. **声明不可破坏的边界**：例如禁止修改 `.github/`，禁止直接操作生产数据库的连接字符串。
2. **预置运行时的上下文**：明确技术栈版本、包管理器、测试框架、常用命令。
3. **标准化代理行为**：定义输出风格（如“不要生成注释掉的 dead code”）、错误处理策略、文件组织规则。

---

## 一个行之有效的 AGENTS.md 结构

下面是在多个 OpenClaw 项目中验证过的模板骨架，你可以按需裁剪：

```markdown
# Project context
- Name: ecommerce-replatform
- Purpose: migrate checkout flow to Go microservices
- Repo: monorepo with apps/ and packages/

## Tech stack & conventions
- Go 1.22, strict module mode
- Node.js 20 (only for legacy admin UI in apps/admin-v1)
- DB: PostgreSQL 15, migrations in db/migrations
- Formatting: gofumpt for Go, biome for TS
- Tests: go test ./... (unit), make integration (integration)

## Critical rules
- NEVER modify or delete files under .github/workflows/
- Do not run database migrations unless explicitly asked
- All new Go packages must include a doc.go
- When generating shell commands, prefer make targets over raw commands

## Common workflows
- Lint: make lint
- Full CI: make ci
- Add a migration: go run cmd/migrate new <name>

## Output preferences
- No placeholder comments like `// TODO: implement`
- When suggesting changes, provide full unified diff
- Group file changes by layer (domain first, then adapters, then handlers)
```

关键原则：**只写会被违反的规则，不要写“正确废话”**。如果项目从不用 Docker，就不必特意声明 “Don’t modify Dockerfile”。

---

## 如何接入 OpenClaw 的运行时

在项目根目录放置 `AGENTS.md` 后，OpenClaw 默认会扫描并注入为代理的系统上下文。可以通过以下方式验证是否生效：

```bash
openclaw agent run "what do you know about this project?"
```

如果代理在回复中引用到了 `AGENTS.md` 里的关键信息（如项目用途、技术栈），说明加载成功。

多模块场景下，可以利用 OpenClaw 的 `context.files` 配置精确控制加载范围，避免将一个巨型 monorepo 的所有 README 一并注入导致上下文污染。示例 `.openclaw.yml`：

```yaml
context:
  files:
    - AGENTS.md
    - apps/checkout/AGENTS.md   # 子模块专属手册
    - docs/adr/                 # 架构决策记录
```

建议每个子模块维护自己的 `AGENTS.md`，全局文件只保留跨模块的规则。

---

## 实践过程中踩过的坑

### 1. 把 AGENTS.md 变成了架构说明文档
过长内容（超过 6000 字符）会显著占用上下文窗口，导致代理对规则的理解精度下降。解决方案是**用短句、用列表、用否定句式**（如 “Do not...”）代替大段描述。对于详细设计文档，通过 `context.files` 按需引用，不要全量注入。

### 2. 规则之间存在隐性冲突
例如同时写了 “Always add unit tests for new functions” 和 “Minimize file changes”。代理可能陷入循环：为了最小化改动而不写测试，随后又被监视工具提示“缺少测试”。需要以优先级或分阶段指令来处理这类冲突，比如 “When adding a new public function, include a test in the same commit; keep other modifications minimal.”

### 3. 缺少版本意识
`AGENTS.md` 变更后，旧的对话会话仍然保留着最初注入的上下文。设计流程时应该明确：在关键规则变更后，要求代理开启新会话；或在自动化流水线中利用 `openclaw session reset` 强制刷新。

### 4. 依赖文件未持久化
有些团队喜欢用脚本动态生成 `AGENTS.md`（例如从 `package.json` 中抽取命令），但忘记将其加入版本控制，导致环境差异。务必让 `AGENTS.md` 成为 Git 追踪的静态文件，必要时通过 pre-commit hook 校验生成内容与当天真实状况一致。

---

## 可复用建议

- **模板化**：为同类项目（如 Go 微服务、React 后台）准备基础模板，通过 OpenClaw 的 `template` 命令初始化。
- **联动 MCP**：如果你的代理通过 MCP 连接了数据库或内部 API，在 `AGENTS.md` 中声明可用的 resource 和 tool 名称，避免代理凭空捏造调用方式。
- **条件规则**：利用 OpenClaw 的 `environment` 字段（staging/production）定制不同规则，比如 “In production, require manual approval before sending HTTP requests to external services”。
- **定期修剪**：每隔几个 sprint 审查一次 `AGENTS.md`，移除不再适用的约束。膨胀的规则比没有规则更危险。

---

## 总结

`AGENTS.md` 的本质不是一份文档，而是你向 AI 代理签发的**可信执行边界**。它让一次性对话中“你懂的”变成可复用、可审计、可演进的结构化协议。对于已经在 OpenClaw 上跑着多个工作流的团队，把它纳入代码评审、新成员 onboarding 的流程里，回报会远超编写它的那十几分钟。

最终目标不是让 AI 依赖手册，而是让手册成为团队共识的自然沉淀——人机共享的约定，才是稳定的协作基线。

---

