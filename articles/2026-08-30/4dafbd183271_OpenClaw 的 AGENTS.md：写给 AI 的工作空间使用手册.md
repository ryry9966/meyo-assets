---
title: OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册
feedId: 35320
source: 综合讨论
publishedAt: 2026-08-30
---

# OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册

## 背景：为什么需要一个“给 AI 看”的工作空间说明

在 OpenClaw 里跑 Agent、接 MCP、做自动化时，一个很常见的麻烦是：每次新建会话，AI 都要重新猜项目结构、命令、环境变量；换个模型或换台机器，行为就完全不一样。AGENTS.md 就是用来解决这个问题的——把工作空间的“使用手册”写进一个约定文件，让 OpenClaw 在进入工作空间时自动读取，作为 Agent 的长期上下文。

它和 README.md 不同：README 是给人看的，AGENTS.md 是给 Agent 看的。内容更偏执行指令、路径、约束和工具边界，而不是项目介绍和安装截图。

## 问题：上下文漂移与工具乱用

没有 AGENTS.md 时，典型问题包括：

- Agent 用错包管理器：项目用 pnpm，AI 默认跑 `npm install`；
- 找不到入口文件，反复 `ls` 或猜路径；
- 不了解已接入的 MCP 工具，要么不调用，要么乱传参数；
- 把测试、构建、迁移命令搞混，产生脏数据或误操作；
- 每次会话都要人工重复解释一遍规则，效率很低。

这些问题通常不是模型能力不够，而是工作空间知识没有沉淀成可复用、可版本化的文件。

## 做法/步骤：怎么落地一个有效的 AGENTS.md

1. 在项目根目录或 OpenClaw 工作空间根目录创建 `AGENTS.md`。
2. 先写“项目概览”：一句话说明这个目录是做什么的，技术栈是什么。
3. 写“目录结构”：只列关键路径，不要列全树，避免浪费 token。
4. 写“常用命令”：开发、构建、测试、启动、数据库迁移等，必须标注在哪个目录下执行。
5. 写“约定与边界”：例如“禁止直接修改 `dist/`”“所有 API 调用必须走 `src/lib/api.ts` 封装”“不要运行带 `--force` 的删除命令”。
6. 写“MCP/插件说明”：如果接入了 MCP server 或 OpenClaw 插件，写清楚工具名称、用途、什么时候用、什么时候不要用。
7. 最后写“注意事项”：环境变量、敏感信息、已知坑。

示例片段：

```markdown
# AGENTS.md

## 项目概览
- Next.js + TypeScript 全栈项目，使用 pnpm 管理依赖。

## 常用命令
- 安装依赖：pnpm install（在根目录执行）
- 启动开发：pnpm dev
- 运行测试：pnpm test -- --run

## 约定
- 不要直接修改 prisma/schema.prisma 后自动生成迁移，先运行 pnpm db:check。
- 所有新增页面放在 app/（使用 App Router）。

## MCP
- 只有需要查数据库时使用 database-mcp；写操作前必须人工确认。
```

保存后，OpenClaw 会话进入该工作空间时会加载它，Agent 可以按手册执行，而不是每次重新摸索。

## 踩坑点

- **文件太长**：AGENTS.md 超过 200-300 行后，Agent 可能忽略后半部分，或者占用过多上下文。建议控制在 150 行以内，细节用链接指向 docs。
- **命令过时**：AGENTS.md 里写了 `npm run build`，但项目已经切到 `pnpm build`，Agent 会继续按旧命令执行。每次改构建工具、目录结构后必须同步更新。
- **敏感信息泄露**：不要把 API key、密码、内网地址写进 AGENTS.md，因为它可能被模型读取并出现在日志或调试输出中。
- **只有约束没有原因**：如果只写“不要运行 X”，Agent 有时不理解为什么，换个说法就绕过去了。简单补一句原因，能提高遵守率。
- **路径歧义**：只写 `npm test` 不写目录，Agent 可能在子目录执行失败。明确“在根目录执行”或“在 packages/web 执行”。
- **与 CLAUDE.md 冲突**：如果项目里同时有 CLAUDE.md 和 AGENTS.md，Agent 可能两套都读，规则打架。建议只保留一个主入口，或者在其中注明优先级。
- **期望过高**：AGENTS.md 不是硬约束，Agent 仍可能忽略。重要操作最好配合 OpenClaw 的权限控制或人工确认。

## 可复用建议

- 做一个团队级模板：把通用命令、环境要求、目录约定抽成模板，新项目复制后改几个字段。
- 分层管理：全局级 AGENTS.md 放通用规则，例如“所有项目默认使用 pnpm”；项目级放具体命令和结构。
- 在 prompt 中显式引用：如果 OpenClaw 没有自动加载，可以在系统提示或会话开头写“请先阅读 AGENTS.md 并遵守其中的约束”。
- 定期 review：把 AGENTS.md 当成代码的一部分，提交到版本库，随 PR 一起评审。
- 测试可用性：新建一个会话，只给一个简单任务，例如“跑一下测试”，看 Agent 是否按手册执行，而不是自己猜。

## 总结

AGENTS.md 的本质，是把你对项目的隐性知识变成 Agent 可以在每次会话中复用的显性手册。它不能解决所有问题，但能显著减少重复解释、降低误操作概率，尤其适合长期维护的 OpenClaw 工作空间。写的时候要克制：只写 Agent 真正需要知道的，保持短小、准确、可执行。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/cbc86ad26f0b0315.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/09593205e5e6708f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/ad51b7e5f5a03808.png)

