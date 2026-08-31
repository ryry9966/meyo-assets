---
title: OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册
feedId: 35600
source: 综合讨论
publishedAt: 2026-09-01
---

## 为什么需要 AGENTS.md

在 OpenClaw 这类 Agent 执行环境里，模型每次进入工作空间都像新来的外包工程师：它知道通用编程知识，但不了解你的项目约定、目录历史、命令别名、雷区。没有上下文时，Agent 会做出“合理但错误”的决策：运行错误的测试命令、修改不该动的生成文件、或者用一套全新目录结构重写你的代码。

AGENTS.md 解决的就是这个问题：它是一份写给 AI 的工作空间使用手册，放在项目根目录，Agent 启动或进入目录时读取，从而把团队约定和项目知识前置注入到上下文中。

## 常见问题：没有 AGENTS.md 的 Agent 有多不可控

- **命令乱跑**：Agent 不知道项目使用 pnpm 还是 yarn，只能试错，浪费 token 和时间。
- **目录误判**：把 `dist/` 当源码目录，或者以为 `scripts/` 是临时脚本而随意修改。
- **触碰雷区**：直接修改 `package-lock.json`、数据库迁移文件、或带有 `GENERATED` 标记的文件。
- **重复造轮子**：明明项目里已有工具函数，Agent 自己重新实现一套。

这些问题本质不是模型能力差，而是缺少项目级的“常识注入”。

## 做法：写一份可执行的 AGENTS.md

### 1. 位置与加载

在项目根目录创建 `AGENTS.md`。OpenClaw 会在 agent 会话开始时自动读取该文件，并将其内容拼接到 system prompt 或工具描述中。保持文件在 100-200 行以内，太长会被截断或稀释注意力。

### 2. 推荐结构

一份工程化 AGENTS.md 通常包含以下段落：

```markdown
# Project: xxx

## Overview
一句话说明项目用途和技术栈。

## Directory Layout
- `src/`：源码，不要修改 `src/generated/`
- `tests/`：测试，使用 vitest
- `scripts/`：一次性脚本，可修改
- `dist/`：构建产物，不要手动编辑

## Commands
- 安装依赖：`pnpm install`
- 开发：`pnpm dev`
- 构建：`pnpm build`
- 测试：`pnpm test -- --coverage`

## Rules
- 不要修改 `package-lock.json` 或 `pnpm-lock.yaml`
- 不要提交密钥，密钥通过环境变量注入
- 修改数据库 schema 前先查看 `docs/schema.md`
- 新增依赖前先检查是否已有类似包

## Environment
- 本地开发使用 `.env.local`，不要提交
- 测试环境变量见 `tests/setup.ts`
```

### 3. 编写原则

- **短句、祈使句**：Agent 更容易遵循“不要 X”而不是“请注意尽量避免 X”。
- **明确文件路径**：不要写“构建产物”，写“`dist/` 目录”。
- **区分“建议”和“强制”**：用 `MUST` / `SHOULD` / `MAY` 标记，或单独一个 `## Critical` 段落。
- **保持更新**：AGENTS.md 过时比没有更危险，因为 Agent 会信任它。

## 踩坑点

1. **文档过长**：超过 300 行后，Agent 容易忽略后面的规则。把详细说明放到 `docs/`，AGENTS.md 只写索引和硬约束。
2. **包含敏感信息**：不要把真实密钥、内网地址、客户数据写进 AGENTS.md，它会被提交到仓库并被所有 agent 会话读取。
3. **规则互相矛盾**：例如一边写“不要修改测试文件”，一边写“修复所有失败测试”。矛盾会让 Agent 随机选择一条执行。
4. **过度约束**：写了几十条“不要”，Agent 可能因规则冲突而拒绝执行任何操作。只保留真正重要的约束。
5. **忽略更新**：项目重构后目录变了，但 AGENTS.md 没同步，导致 Agent 按照旧路径操作。

## 可复用建议

- **模板化**：为不同类型的项目准备 AGENTS.md 模板，新项目开箱即用。
- **与 MCP/插件配合**：AGENTS.md 写静态约束，动态信息（如当前分支、CI 状态）让 Agent 通过 MCP 工具查询，避免文档膨胀。
- **纳入 Code Review**：AGENTS.md 的修改也要走 PR，防止个人随意添加私有偏好。
- **版本标记**：在文件末尾加 `Last updated: 2025-XX-XX`，或用 git blame 追踪。
- **团队共识**：写之前先收集团队最想让 Agent 知道的 10 件事，而不是一个人拍脑袋。

## 总结

AGENTS.md 是 OpenClaw 工作空间里成本最低、回报最高的治理手段之一。它不替代代码规范文档，而是把关键约束压缩成 Agent 启动就能读懂的“项目说明书”。写一份好的 AGENTS.md 不是一次性的，需要随着项目演进持续维护。当你发现 Agent 开始“懂事”地使用正确命令、避开雷区时，这份文档就值回了它的维护成本。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/8f426143abb88867.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/dbd025c4838c41b4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/1aa9e53a4498960c.png)

