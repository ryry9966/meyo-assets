---
title: OpenClaw 的 AGENTS.md：给 Agent 写一份它真会读的工作空间手册
feedId: 31865
source: 综合讨论
publishedAt: 2026-08-06
---

## 背景

在 OpenClaw 的日常使用里，Agent 并不是一个“全知全能”的助手——它每次启动都需要重新理解项目上下文。文件结构、测试命令、可用的 MCP 服务器、插件约定、甚至你习惯的 Git 提交风格，这些信息如果只靠临时 Prompt 传递，很快就会变成“每次都说、每次还忘”的循环。

OpenClaw 提供了一种更工程化的方式：在项目根目录放置一个 `AGENTS.md` 文件，把它作为 Agent 工作空间的唯一事实来源。你可以把它理解为写给 AI 的 README——只不过读者不是人，而是一个随时会读取、解析并遵循该文件的 Agent。

这篇文章不会讲“如何写出完美 Prompt”，而是从一个实际维护者的角度，拆解 `AGENTS.md` 的编写、集成、陷阱和可复用模式。

---

## 问题：为什么光靠系统 Prompt 不够

一个典型的 OpenClaw 项目可能涉及：

- 多个 MCP 服务器（如文件系统、数据库、第三方 API）
- 自定义插件或脚本
- 特定的代码风格、测试框架、部署命令
- 团队协作约定（如提交信息格式）

如果我们把这些信息全部塞进 OpenClaw 的全局系统提示里，会有三个明显问题：

1. **跨项目污染**：A 项目的约定会泄漏到 B 项目中，Agent 给出的建议可能完全不符合当前上下文。
2. **维护成本高**：每次新增 MCP 服务器或修改项目结构，都需要同时更新系统提示，很容易遗漏。
3. **无法版本化**：系统提示通常不在代码仓库里，而项目约定必须随代码演进，版本控制是基本要求。

`AGENTS.md` 解决的就是这件事：让项目自己“告诉” Agent 它需要知道的一切，并且这份说明书和代码一起被版本化、一起被评审。

---

## 做法：一份最小化的 AGENTS.md 应该包含什么

我总结了一个实用结构，没必要面面俱到，但要覆盖 Agent 最常“犯错”或“不知道”的信息。

### 1. 项目一句话描述与边界
```
# Project: OpenClaw Demo Workflow
This is a demo project to test automated issue triage with GitHub MCP.
Agent should NOT modify any .env files or files outside /src.
```
明确边界能防止 Agent 在探索工作空间时误操作敏感文件。

### 2. 可用工具与 MCP 服务器
用表格列出可用的 MCP 服务器及其用途，而不是让 Agent 自己从配置中猜测。
```markdown
| MCP Server      | Purpose                         | Notes                 |
|-----------------|---------------------------------|-----------------------|
| github          | Issue/PR management             | Read-only on prod repos |
| sqlite          | Local bug DB                    | Path: ./data/bugs.db  |
| custom-plugin   | Internal deploy triggers        | See /plugins/deploy   |
```
如果某个 MCP 服务器仅在特定任务下才启用，也要标注清楚。

### 3. 常用命令与脚本
Agent 经常需要执行 lint、测试、构建等操作。把命令写在 `AGENTS.md` 里，可以避免它自己编造命令。
```markdown
## Commands
- Lint: `pnpm lint`
- Test: `pnpm test -- --coverage`
- Build: `pnpm build`
- Dev server: `pnpm dev`
```
如果是 monorepo，建议标注每个命令的执行目录。

### 4. 项目结构速览
不需要完整目录树，只用列出对 Agent 最重要的路径：
```markdown
## Key paths
- Source: `src/`
- Tests: `tests/`
- Agent workspace: `.openclaw/`
- Output artifacts: `dist/`
```

### 5. 约束与风格约定
Agent 生成的代码、提交信息、注释等都要符合你的团队规范。把关键规则写成清单：
```markdown
## Conventions
- TypeScript strict mode, no `any`
- Git commits follow conventional commits
- Comments must be in English
- Error messages should include a unique error code
```

---

## 集成：让 OpenClaw 真正读取 AGENTS.md

OpenClaw 默认不会自动加载 `AGENTS.md`，我们需要显式配置。通常有两种方式：

1. **在 OpenClaw 的系统提示中引用**  
   把文件内容通过变量注入到系统提示的顶部，例如：
   ```markdown
   You are an AI agent in project: {{PROJECT_NAME}}.
   Read the following workspace manual from AGENTS.md:
   
   {{AGENTS_FILE_CONTENT}}
   ```
   然后利用 OpenClaw 的环境变量或模板功能，在启动时读取并替换 `{{AGENTS_FILE_CONTENT}}`。

2. **使用 OpenClaw 的 Workspace Context 特性**（如果版本支持）  
   某些 OpenClaw 版本允许在 `openclaw.yaml` 里指定一个 `contextFile`，并在每次会话开始时自动注入。这是最省心的方式，但需要确认你的版本是否支持。

我更推荐第一种，因为它可控、可调试。你可以在非生产环境下先用简单的 shell 脚本验证 Agent 是否正确理解了文件内容。

---

## 踩坑点

实践中几个容易栽跟头的地方：

- **文件太长，Token 消耗严重**  
  一个 2000 行的 `AGENTS.md` 会让 Agent 的处理速度明显下降，且容易出现“丢失中间信息”的问题。保持在 300 行以内，多用链接指向外部文档。
  
- **Agent 选择性忽略**  
  如果 `AGENTS.md` 里同时有“测试命令”和“不要修改 .env”，Agent 可能在执行复杂任务时只记住前者而忽略后者。关键约束要放在文件最前面，并用 **粗体** 或 `⚠️` 标记。
  
- **没有与系统提示的优先级约定**  
  如果系统提示说了“你可以自由修改文件”，而 `AGENTS.md` 里写了“禁止修改 .env”，Agent 可能以系统提示为准。需要在两者中明确优先级：`AGENTS.md` 作为项目级约束应当覆盖通用指令。
  
- **更新滞后**  
  项目结构变了、MCP 服务器换了，但 `AGENTS.md` 没有同步更新。Agent 就会拿着过时的手册干活，后果比没有手册更严重。建议在 CI 里加一个检查：当某些关键文件（如 `openclaw.yaml`）变更时，提醒更新 `AGENTS.md`。

---

## 可复用建议

基于上面这些坑，我提炼了几个可复制的做法：

- **模板化但不僵化**：维护一个公司级的 `AGENTS.md` 模板，但每个项目只需填写“项目特定信息”部分，其他部分保持最小。
- **分层信息**：第一层是 Agent 每次都必须遵守的核心约束（50 行以内），第二层是具体工具、命令，第三层指向外部文档链接。
- **验证闭环**：在 Agent 完成关键任务后，让它在返回结果中附带一句“已遵守 AGENTS.md 中的 X 约束”，这是一种轻量级的自我检查机制。
- **与 Onboarding 结合**：当新开发者加入项目时，让他们先读懂 `AGENTS.md`，因为这也是 AI Agent 会遵循的规则。人能理解，Agent 才更可能正确执行。

---

## 总结

`AGENTS.md` 不是一个花哨的功能，而是一个典型的基础设施薄层。它解决的是多项目、多工具、多约定环境下，Agent 上下文管理的可维护性问题。

把它写好，意味着你不再需要在每次会话中重复解释项目背景；把它版本化，意味着 Agent 的“知识”可以随代码库一起演化；把它做薄，意味着它不会成为新的维护负担。

对于已经用上 MCP、插件和自动化任务流的团队来说，一份扎实的 `AGENTS.md` 比精心调参的系统 Prompt 更可持续。Agent 需要的不是一个万能的父类提示，而是一个能随叫随到的项目手册。

---

