---
title: OpenClaw 的 AGENTS.md：用结构化上下文驯服你的 AI 代理
feedId: 31758
source: 综合讨论
publishedAt: 2026-08-05
---

# OpenClaw 的 AGENTS.md：用结构化上下文驯服你的 AI 代理

## 背景：当 AI 代理踏进你的代码库

在 OpenClaw 的 Agent 工作流里，我们习惯用自然语言写 prompt 来驱动大模型，让它去调 MCP 工具、读写文件、执行 Shell 命令。但很快你会发现一个尴尬的事实：你的代码库在 AI 眼里只是一个扁平的目录树，它既不知道哪些模块是核心域，也不清楚你刚刚重构过数据层，更不会意识到 `main` 分支上的 CI 近期炸了是因为某个环境变量改了名。

于是你会在每一次会话开头反复粘贴同样的项目介绍，贴到厌烦。或者更糟——AI 基于过期认知生成了一大段看起来合理但完全走偏的代码，你不得不花一个下午修正。

OpenClaw 给出的解法之一就是 **AGENTS.md**。它不是简单的系统提示词，而是一份**工作空间使用手册**，专门写给 Agent 看。Agent 在进入工作空间时自动加载它，形成持续、一致的项目认知。

## 问题：上下文缺失如何污染 Agent 行为

在一个真实的自动发布流水线里，我们配置了一个 OpenClaw Agent 负责从 Issue 提取需求、生成 PR、触发 MCP 调用执行 CI。但上线第二周就暴露出三个典型问题：

1. **命令滥用**：Agent 不知道我们禁用了 `npm publish` 的本地执行，反复尝试直接用 MCP 的 `run_shell` 触发发布，导致安全策略告警。
2. **路径假设**：项目使用 pnpm monorepo，结构是 `apps/web` 和 `packages/shared`，但 Agent 总是把依赖装到根目录，因为没人告诉它 `npm install` 应该在子包目录执行。
3. **规则丢失**：团队约定所有 API 路由文件必须包含 Swagger 注释块，Agent 生成的代码完全没有，Review 被打回三次。

这些都不是模型能力问题，而是**上下文缺失**。Agent 需要一个能被机器稳定解析、又足够灵活让人类维护的载体。

## 做法：如何编写一份工程化的 AGENTS.md

OpenClaw 的 AGENTS.md 放在项目根目录，Agent 在每次会话初始化时读取。下面是一个精简但实用的结构：

```markdown
# Workspace: oneflow-backend
type: monorepo
language: TypeScript (strict), Node.js 20
package_manager: pnpm@9

## Context
- 这是 OneFlow 的后端服务群，负责工作流编排、触发器调度和 API 网关。
- 最近一次重大变更：2025-06-03 将消息队列从 BullMQ 迁至 RabbitMQ，相关适配器在 `packages/broker`。
- CI 状态：`main` 分支当前红色，原因是缺少环境变量 `RABBITMQ_VHOST`，待修复。

## Workspace Map
- `apps/api`: REST API 服务，端口 3001
- `apps/worker`: 任务执行器，依赖 Postgres 和 Redis
- `packages/shared`: 跨服务类型定义与工具函数
- `scripts/`: 本地开发辅助脚本，**禁止在 Agent 中直接调用**（涉及数据库重置）

## Rules
1. **禁止执行 `pnpm publish` 或任何 npm publish 操作**，发布由 CI 流水线接管。
2. 所有新增 API 路由必须在文件顶部添加 Swagger JSDoc 注释块。
3. 修改数据库 schema 前，必须先输出变更计划供人工确认，不得直接执行 `prisma migrate deploy`。
4. 安装依赖：如果是 monorepo 子包，使用 `pnpm --filter <package>` 方式安装。

## Available Tools (MCP)
- `run_shell`: 仅限非破坏性命令，不允许 `rm -rf`、`git push --force`。
- `read_file`, `write_file`: 标准文件操作。
- `github`: 创建 PR、添加 Review comment。
- `postgres`: 仅允许只读查询。

## Dynamic Hooks
- 在每次 `write_file` 后，自动运行 `pnpm lint --fix` 在对应文件路径。
```

这份文件的核心在于 **可执行、不可协商**。Agent 会把它当成硬约束。

同时我们利用 OpenClaw 的上下文模板功能，将一部分动态信息抽成变量，比如 `{{CI_STATUS}}` 由 pre-session hook 注入，避免人工手动更新。

## 踩坑点

1. **AGENTS.md 变成垃圾桶**
团队一开始把所有想“告诉 AI”的东西都往里塞，包括历史决策理由、未收敛的架构讨论、个人代码风格偏好。文件膨胀到 800 行，Agent 读取消耗大量 token，反而忽略了核心规则。我们后来硬性规定：只保留与 Agent **行为约束**直接相关的内容，其余放到 docs/ 或项目 wiki。

2. **规则与 MCP 工具描述冲突**
MCP server 提供的工具描述里写着 `run_shell` 可以“execute any command”，但 AGENTS.md 禁止了若干命令。Agent 有时会倾向于相信 MCP 的工具描述，因为它更“新鲜”。解决办法是在 MCP 工具注册时加一层 wrapper，将受限命令直接移除，而不是只靠文档约束。

3. **动态数据过时**
手动维护的上下文片段（比如 CI 状态）一天后就不再准确。Agent 会基于错误认知做决策。通过 hook 自动注入或用 OpenClaw 内置的 context provider 拉取实时状态，可以解决此问题。

4. **多 Agent 共享 AGENTS.md**
当一个 workspace 被多个 Agent 共同维护（比如一个负责代码生成，另一个负责 Code Review），AGENTS.md 里的约束会出现面向角色的细微差异。我们引入 `## Role: code-generator` 和 `## Role: reviewer` 条件块，让 Agent 只加载自己角色的章节，减少干扰。

## 可复用建议

- **用结构而不是长段落**：Agent 对 Markdown 标题、列表、代码块的解析明显优于白话长文。把规则写成 list，路径用 tree 表示，禁止项用粗体和 `code` 标注。
- **定期做“压缩”**：每两个迭代回顾一次 AGENTS.md，删除已不再适用的规则，合并重复项。可以将它视为代码资产，纳入 PR Review 范围。
- **和 MCP 能力对齐**：AGENTS.md 里引用的每个工具名必须和 MCP manifest 中的完全一致，否则 Agent 会出现工具幻觉。
- **提供逃生舱**：允许 Agent 在不确定时直接提问，而不是硬猜。比如加上一句：“当规则与任务冲突或模糊时，暂停执行并向用户请求澄清。”
- **分层管理**：如果项目很大，考虑拆分成 AGENTS.root.md、AGENTS.api.md 等，通过 OpenClaw 的 workspace 配置按需加载。

## 总结

AGENTS.md 本质上是一个将团队共识工程化、并输送给 AI 的契约文件。它不能替代 prompt engineering，但能大幅减少你每次向 Agent 解释“我们这个项目哪里不一样”的成本。写好它，就像给新来的同事准备一份靠谱的 Onboarding Doc——只是这个同事记忆力超群，但特别缺乏常识，需要你把边界画得无比清晰。

随着 OpenClaw 生态里 MCP 工具和插件越来越多，AGENTS.md 会成为 Agent 行为的最后一道防波堤。把精力花在维护这份手册上，比反复救火式的修复 AI 生成的低质量代码要划算得多。

---

