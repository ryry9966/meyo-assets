---
title: OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册
feedId: 35037
source: 综合讨论
publishedAt: 2026-08-28
---

OpenClaw 的 Agent 进入工作空间时，默认只能通过目录扫描、读取部分文件和工具试探来理解环境。项目小的时候还行，但一旦涉及多个服务、MCP 工具、插件脚本和自动化任务，这种“现场推理”就会迅速暴露问题：Agent 可能跑错包管理器、在错误目录执行命令、把构建产物当源码修改，或者反复问“这个目录是干什么的”。

AGENTS.md 要做的事情，就是把这些“人脑里知道、但 AI 每次都要重新猜”的信息，落成一份稳定的工作区说明书。OpenClaw 在工作区根目录读取它，作为任务执行前的上下文注入。它不替代 README，也不负责解释业务背景，只回答一个问题：在这个工作区里，AI 应该怎么干活。

## 一个最小可用的 AGENTS.md

建议直接放在项目根目录，用 Markdown 写。结构不用复杂，关键是目录职责、命令、边界和验证方式。

```markdown
# AGENTS.md

## 项目地图
- `services/api/`：主 API 服务，Node.js + Fastify
- `services/worker/`：后台任务，通过 Redis 消费队列
- `mcp/`：自定义 MCP 工具，改动后必须重新构建
- `deploy/`：部署与回滚脚本

## 常用命令
- 安装依赖：`pnpm install --frozen-lockfile`
- 启动 API：`cd services/api && pnpm dev`
- 运行测试：`pnpm test -- --runInBand`
- 构建 MCP：`pnpm build:mcp`

## 约束
- 不要直接修改 `dist/` 或 `build/` 下的文件
- 不要在根目录创建临时文件，统一放 `.scratch/`
- 修改数据库结构前，先生成迁移文件并等待用户确认

## 验证
- 改完 API 后运行：`pnpm test:api`
- 改完 MCP 后运行：`pnpm build:mcp && pnpm test:mcp`
```

这套结构的作用是让 Agent 在动手前先知道边界。比如“不要直接改 `dist/`”可以避免大量无效 diff；“数据库迁移前等待确认”则把高风险操作拦在任务开始前。

## 踩坑点

1. **不是越长越好。** AGENTS.md 写太长会占用上下文窗口，Agent 反而忽略关键约束。建议控制在 300 行以内，详细设计文档用链接指向 `docs/`。
2. **命令不要写“合理猜测”。** 如果命令是错的，Agent 会忠实执行错误命令。宁可写“运行测试前先确认 `package.json` 中的 `test` script”，也不要凭空写一个不存在的命令。
3. **约束要带边界或原因。** 只写“不要乱动”没有用，写“不要直接改 `.env`，所有环境变量通过 `scripts/set-env.sh` 写入”更可执行。
4. **子目录规则冲突。** 如果大型项目里某个子服务需要单独说明，可以在子目录放一个轻量 AGENTS.md，但根目录文件要写明“子目录规则见各自 AGENTS.md”，避免 Agent 混淆。
5. **更新滞后比没有更危险。** CI 命令、目录结构变化后，AGENTS.md 必须同步更新，否则会误导后续所有任务。

## 可复用建议

- 把 AGENTS.md 纳入版本控制，和代码一起 review。
- 用“允许/禁止”列表替代大段描述；Agent 对列表的响应比段落更好。
- 如果 OpenClaw 配了 MCP 工具，在 AGENTS.md 里写清楚哪个 MCP server 负责哪类操作，避免工具误用。
- 高风险操作单独列一个“需要人工确认”区块，例如发布、删除、数据库迁移。
- 每两周跟着 CI 或目录变更检查一次 AGENTS.md 是否还准确。

## 总结

AGENTS.md 不是给人类看的 README，它是 Agent 的“标准操作程序”。对 OpenClaw 用户来说，它的价值不在内容多，而在于减少每次任务里重复的“工作区解释”和“误操作修复”。从一份 50 行的最小模板开始，把最重要的目录、命令、边界写进去，比写一篇长文档有效得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/4d78c876d3b2546e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/95b130f3e915eca8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/390d52f82b9cd1c5.png)

