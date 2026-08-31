---
title: OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册
feedId: 35489
source: 综合讨论
publishedAt: 2026-08-31
---

# OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册

在 OpenClaw 里，AGENTS.md 不是 README，也不是系统 prompt 的替代品。它更接近一份放在工作空间里的“使用手册”：告诉每次新启动的 agent，这个目录怎么跑、哪些不能碰、工具和 MCP 应该怎么选。

背景是，OpenClaw 工作空间通常不只是聊天，还会挂浏览器插件、文件 MCP、命令行执行和自动化任务。每次任务开始时，agent 如果没有持久上下文，就只能靠短时记忆和试探。常见问题很具体：测试命令猜错、生成目录被手改、MCP 越过工作空间读到家目录、危险命令没人拦。临时纠正一次只解决一次，下一次启动又忘。

AGENTS.md 的价值，就是把这类“工程约定”固化成随项目走、可版本化、可 review 的上下文。建议不要把它写成大而全的文档，而是写成“可执行的护栏”。

## 做法/步骤

1. 在工作空间根目录新建 `AGENTS.md`。
2. 先写四块：目录边界、命令入口、工具/插件选择、禁令。
3. 用短句和列表，不写散文。

一个可用的骨架如下：

```markdown
# AGENTS.md

## Workspace map
- `src/` 运行时代码，`tests/` pytest，`scripts/` 一次性迁移，`docs/` 对外文档
- 不要修改 `generated/`，该目录由 `pnpm codegen` 生成
- `.env` 只读，不回写

## Commands
- install: `pnpm install --frozen-lockfile`
- test: `pnpm test -- --runInBand`
- lint: `pnpm lint`
- 禁止在根目录直接执行 `ts-node`，统一走 `pnpm exec`

## Tools & plugins
- 浏览器自动化仅使用 OpenClaw 内置 browser 插件
- 文件搜索优先 `rg`，不使用 `find`
- MCP `filesystem` 根目录限定为当前工作空间，不访问 `~/.ssh`、`~/.aws`

## Guardrails
- 禁止执行 `git push --force`
- 禁止删除 `.cache/`，清理统一走 `pnpm clean`
- 修改 schema 前先运行 `pnpm db:check`
- 涉及超过 3 个文件的改动，先给方案，不直接改
```

写完骨架后，把每次对话中人工纠正过的规则回写进去。比如 agent 错误执行了 `rm -rf .cache`，就补一句“禁止删除 `.cache/`，清理走 `pnpm clean`”。这样 AGENTS.md 会随实际使用逐渐变准。

如果项目变大，不要在子目录复制整份 AGENTS.md；根文件保持全局规则，子目录只放局部例外。长任务开始前，可以先让 agent 用一句“我理解的三条关键约束是……”复述，以确认它真的读到了，而不是只在上下文里“见过”。

## 踩坑点

- **太长**：超过 80 行后，模型对中后段约束的遵循度会明显下降。
- **模糊词**：`尽量`、`建议` 会让 agent 不确定；必须使用“必须/禁止”。
- **和真实脚本不一致**：AGENTS.md 里的 test 命令要和 CI 对齐，否则 agent 按文件执行，反而破坏 CI。
- **权限开太大**：文件系统 allow all 后，目录边界基本失效。
- **写成 README**：应该给命令、路径和边界，而不是写设计感想或模块说明。
- **只增不删**：过期规则会污染决策，建议每次迭代时清理一次。

## 可复用建议

- 保持 20-40 行；超过 80 行就拆分清理。
- 命令写完整，例如 `pnpm test -- --runInBand`，不要只写 `pnpm test`。
- 危险命令用黑名单明确列出：`git push --force`、`rm -rf`、`db reset`。
- MCP 工具统一加根目录限制，避免跨工作空间访问。
- 把 AGENTS.md 纳入 PR review，改动规则时像改代码一样说明原因。
- 多写“事实”，少写“观点”；agent 更容易执行确定性的指令。

## 总结

AGENTS.md 是 OpenClaw 工作空间里的操作边界和速查表。它不负责提升模型“智力”，但能显著减少 agent 在长链路任务中的错误率和来回解释成本。短、准、可执行，比写得全更重要。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/4aebf6b8ce2e267f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/331a1864798bba5e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/dbadbddd24ac8344.png)

