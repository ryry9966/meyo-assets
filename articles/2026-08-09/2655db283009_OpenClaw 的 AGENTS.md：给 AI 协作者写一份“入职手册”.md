---
title: OpenClaw 的 AGENTS.md：给 AI 协作者写一份“入职手册”
feedId: 32205
source: 综合讨论
publishedAt: 2026-08-09
---

## 为什么需要一份“AI 工作空间使用手册”

在 OpenClaw 生态里，AI 代理（Agent）被越来越多地用来执行代码生成、文件操作、命令调用甚至多步自动化任务。无论是通过 MCP 工具链串联本地开发环境，还是用插件扩展代理的能力，我们都在尝试让 AI 像一个真实的协作者那样工作。但一个很现实的问题迅速暴露出来：**代理缺少关于当前项目的上下文**。

通常这些信息散布在 README、内部 Wiki、团队约定或不言自明的开发经验里。一个典型的例子是，你用代理生成一个数据库迁移脚本，它生成了符合语法却无视你项目命名规范的结果；或者它反复使用你早已废弃的构建命令，因为 system prompt 里没法把所有细节都实时同步进去。传统做法——每次任务前在 prompt 里写大段说明——既低效又容易被遗忘。

在 OpenClaw 的实践中，**AGENTS.md** 就是为解决这个“上下文对齐”问题设计的。它是一份放在项目根目录下的 Markdown 文件，作为代理启动时自动加载的“工作空间使用手册”。代理会优先参考其中的约束和指南，从而显著降低行为的随机性。

## AGENTS.md 应该怎么写

AGENTS.md 不是 README 的缩略版，也不是技术文档的合集。它的读者是 AI 模型，因此**格式和措辞需要为模型解析优化**。经过多个项目的反复尝试，我整理出一套较为有效的结构：

### 1. 项目简短概要（2‑3 句）
直述项目是做什么的，使用的主要技术栈。目的是让代理快速建立领域认知，不深入解释业务，避免 noise。

```markdown
## Project Overview
A TypeScript-based CLI tool for managing cloud infrastructure. Uses React for the dashboard.
Key stack: Node 20, pnpm, Prisma, PostgreSQL.
```

### 2. 工作区目录结构（精简版）
只列出代理经常需要手动操作的关键路径，比如源代码入口、测试目录、配置文件位置。不要 dump 完整目录树，模型在长上下文下会稀释注意力。

```markdown
## Directory Map
- `src/commands/` — CLI command implementations
- `src/utils/`   — shared helpers (do not add business logic here)
- `tests/`       — vitest integration tests
- `prisma/`      — database schema & migrations
```

### 3. 环境与常用命令
明确列出开发中反复使用的命令，注明环境变量需求。用代码块包裹，方便模型引用。

```markdown
## Commands
- Install: `pnpm install`
- Dev server: `pnpm dev` (requires `.env` file; see `.env.example`)
- Run tests: `pnpm test` (add `-- --coverage` for reporting)
- Migrate DB: `pnpm prisma migrate dev`
```

### 4. 编码约定与显式约束
这里是最容易踩坑，也最能提升代理行为质量的部分。我习惯将规则分为“禁止项”和“推荐做法”，并用极其直白的指令写出，例如：

```markdown
## Rules
- NEVER use `any` in TypeScript; always derive types from schema or explicit interfaces.
- Do not generate files outside `src/` unless explicitly asked.
- Prefer `async/await` over raw promises.
- Always use `logger` from `src/utils/logger` instead of `console.log`.
- Database transactions must be explicit and rolled back on test teardown.
```

注意：规则描述用祈使句加“NEVER”“MUST”这种强语气词效果更好，模型倾向于更严格地遵循。

### 5. 可用的工具/插件说明（OpenClaw 特有）
如果你的 OpenClaw 环境连接了特定的 MCP 服务器或内部插件（比如文件系统、Git 操作、自定义 API），可以在 AGENTS.md 里声明代理可以调用的工具列表及其约束。这能避免代理幻想出不存在的能力。

```markdown
## Available Tools
- `filesystem` — read/write within project root only
- `git` — use `git log --oneline` for history, never force-push
- `mcp_db` — read-only queries to `analytics` database
```

### 6. 任务示例（正例 + 反例）
提供一两个简短的交互示例，可以极大减少行为偏差。例如：

```markdown
## Examples
✅ Good task: "Create a new CLI command in `src/commands/` that accepts a `--env` flag. Write unit test with mocked DB."
❌ Bad task: "Fix everything." (too vague)
```

## 实践中的常见坑

1. **把 AGENTS.md 写成百科全书**  
   文件过长（超过 200 行）会导致模型只关注开头或结尾，中间规则被忽略。保持克制，用链接指向详细文档。

2. **包含敏感信息**  
   切忌在 AGENTS.md 里写入密钥、token 或内网 IP。代理可能在不安全通道泄漏这些内容，或把文件内容意外嵌到生成输出中。

3. **规则互相矛盾或过时**  
   团队演进后，旧的构建命令或 deprecated 的规范若没有及时更新，代理会严格按照过时说明执行，酿成更隐蔽的错误。建议每次修改项目关键流程时同步审查 AGENTS.md。

4. **不测试即交付团队**  
   写完 AGENTS.md 后，至少让代理执行一个简单、可预测的任务（如“列出项目中的测试文件路径”），观察它是否遵循规则。很多时候模型“读”了却未必“照做”，需要调整措辞或补充强制指令。

5. **忽视模型能力边界**  
   不要期望一份 AGENTS.md 能让代理完全理解复杂的业务领域模型。大范围的业务上下文仍然需要结合 session 内的 prompt 微调。

## 可复用的工程化建议

- **纳入版本控制，并与 CI 联动**  
  将 AGENTS.md 作为项目元数据的一部分，通过 lint 工具检查其结构是否包含最小必要章节。一旦有破坏性修改，自动提醒更新 AGENTS.md 对应部分。
- **为多代理场景添加标签**  
  如果你的项目中不同的代理负责不同环节（例如一个代理负责写代码，另一个负责审查），可以在 AGENTS.md 中使用条件标记，例如 `<!-- AGENT: reviewer -->`，再通过 OpenClaw 的代理选择器加载不同段落。
- **结合 MCP 工具实现动态上下文**  
  当项目依赖版本或配置文件频繁变动时，直接在 AGENTS.md 中维护静态信息容易过时。可以写一个简单的 MCP 工具，在代理启动时抓取当前环境快照（如 Node version、依赖树），然后代理结合这份动态快照和静态 AGENTS.md 做决策，兼顾准确与效率。

## 总结

AGENTS.md 本质上是一种**确定性工程手段**：在 AI 行为不可控的背景下，用结构和明确的约束把变数降至最低。它不是一劳永逸的“魔法文件”，而更像一个需要持续维护和验证的接口定义。当你的 OpenClaw 代理开始表现得像一位真正了解项目约定、减少无谓返工的协作者时，背后往往是这份文档在默默起作用。

与其每次对着代理反复纠正，不如把规则写下来，交给 AGENTS.md。这是一种朴素的工程思维，但对 AI 协作的稳定性提升，远比想象中大。

---

