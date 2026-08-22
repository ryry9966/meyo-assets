---
title: OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册
feedId: 34132
source: 综合讨论
publishedAt: 2026-08-22
---

## 背景

OpenClaw 的工作空间里，目录结构、命令约定、MCP/插件开关、发布流程，每次新会话都要重新交代。稍一偷懒，AI 就会用错包管理器、漏开 MCP、误改测试文件。与其在聊天里反复纠正，不如把这些信息固化成一个工作空间级说明书，让 OpenClaw 进入项目时先读规则。

`AGENTS.md` 就是这份说明书。它不是 README 的替代品，也不是给人类看的项目介绍，而是写给 AI 的约束文件：短、准、可验证。

## 我遇到的问题

在 OpenClaw 里做自动化任务时，主要遇到三类问题：

1. **命令不一致**：AI 有时用 `npm`，有时用 `pnpm`，有时直接 `npx`，导致 lockfile 变动。
2. **MCP/插件状态不明确**：某个 MCP 只应在本地启用，但 AI 在 CI 场景也尝试连接；某个插件只处理 mock 数据，AI 却拿去改真实接口。
3. **边界模糊**：AI 会自动格式化无关文件、删除它认为“无用”的脚本、甚至尝试直接提交代码。

这些问题都指向同一件事：工作空间缺少一份指令明确的 `AGENTS.md`。

## 做法

### 1. 先写最小版本

在项目根目录创建 `AGENTS.md`，只写五块内容：

- 项目一句话用途
- 目录结构与职责边界
- 常用命令，只写可复制的
- MCP/插件清单与启用条件
- 禁止事项

示例：

```markdown
# AGENTS.md

## Purpose
OpenClaw workspace for order-service automation.

## Commands
- Install: `pnpm install --frozen-lockfile`
- Test: `pnpm test --runInBand`
- Lint: `pnpm lint`
- Build: `pnpm build`

## MCP/Plugins
- `mcp-postgres`: enabled locally only, use env `DATABASE_URL`
- `plugin-mock-api`: enabled only under `scripts/mock/`

## Guardrails
- Do NOT edit `packages/legacy/**`
- Do NOT run migrations without asking
- Do NOT change lockfile unless needed
```

这份最小版本可以先解决 80% 的误操作。

### 2. 目录级增量规则

根 `AGENTS.md` 总容易变长。可以把子目录专属规则放到对应目录的 `AGENTS.md` 里，例如 `packages/worker/AGENTS.md` 只写 worker 的构建方式和队列测试要求。OpenClaw 读取时按上下文合并，根文件保持全局规则，子文件只保留增量。

### 3. 规则要强制，不要建议

不要写“尽量使用 pnpm”，要写“必须使用 `pnpm`，禁止使用 `npm`”。模糊表达会被 LLM 忽略。用 `MUST`、`DO NOT`、`ONLY` 这类词，并在规则后给出可验证条件，比如“若 lockfile 被改变，视为规则违规”。

### 4. 把验证写进协作流

让 OpenClaw 在完成任务前检查是否违反 `AGENTS.md`。你可以在任务结尾追加：

```text
Before finishing, verify your steps against AGENTS.md and list any violations.
```

如果团队有 CI，可以用 pre-commit 检查 `AGENTS.md` 是否更新。规则变更也应像代码一样评审。

## 踩坑点

- **写太长**：超过 150 行的根 `AGENTS.md` 会消耗上下文且降低遵循度。尽量控制在 100 行以内，详细说明放 README。
- **把 README 复制进去**：背景、架构演进、业务名词解释对 AI 任务未必有用。AI 需要的是执行指令，不是知识科普。
- **信息过期**：MCP 名称、环境变量、目录结构变了，`AGENTS.md` 没跟着改，AI 会照着旧规则执行。每次改结构时同步更新。
- **敏感信息泄露**：不要把 API key、token、内部地址写进 `AGENTS.md`。统一走环境变量或密钥管理。
- **子目录冲突**：根文件说“禁止改 `scripts/`”，子目录 `scripts/AGENTS.md` 又写“可修改 mock 脚本”。规则冲突会让 AI 选择更容易的那个。需要明确优先级。

## 可复用建议

- 用模板初始化：新项目先放一份 30 行的 `AGENTS.md`，再逐步补充。
- 按“命令、目录、MCP/插件、禁区、验证”五段组织，不写散文。
- MCP/插件区分三档：默认启用、按需启用、禁用。
- 每季度做一次规则审计，删掉过时和重复项。
- 把 `AGENTS.md` 纳入版本管理，并在 PR 里说明规则变化。

## 总结

`AGENTS.md` 的本质是把人的执行约束翻译成 AI 能稳定遵守的工作空间手册。它不需要聪明，只需要每次执行时都看得到、读得懂、可核对。相比每次会话里反复交代，一份维护良好的 `AGENTS.md` 能显著减少 OpenClaw 的低级错误，也能让自动化流程更可预期。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/d633df4656a2ab4e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/c2f33aa389223165.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/cc4faef16ac3f328.png)

