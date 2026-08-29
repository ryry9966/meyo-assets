---
title: OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册
feedId: 35168
source: 综合讨论
publishedAt: 2026-08-29
---

# OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册

在 OpenClaw 里跑自动化、MCP 或多插件任务时，很多翻车不是能力问题，而是上下文漂移：上次会话明确 `pnpm test`，下次它可能执行 `npm test`；你希望产物进 `dist/`，它写到 `scripts/`；一个只读类 MCP 工具被反复调用，甚至触发了写操作。与其每次都靠长 prompt 纠正，不如把工作空间规则固化成 `AGENTS.md`。

## 背景：为什么需要 AGENTS.md

OpenClaw 启动工作空间时，会把根目录的 `AGENTS.md` 作为长期上下文入口。它和普通 README 不同：README 是给人看的项目说明，AGENTS.md 是给 AI 看的操作契约。它要回答的不是“这个项目是什么”，而是“在这个目录里，你能做什么、不能做什么、先做什么”。

没有这份文件，OpenClaw 只能依赖模型猜测目录约定和工具边界；有 MCP/插件以后，猜测的成本会被放大。

## 问题：没有 AGENTS.md 的典型症状

1. 命令选择不稳定：同一仓库一会跑 `yarn`，一会跑 `npm`。
2. 目录写入混乱：生成文件落在根目录、临时目录或源码目录。
3. 危险操作无拦截：模型为了完成目标，可能执行删除、强推、覆盖配置等操作。
4. MCP 权限被滥用：能读的接口被当成能写，能查的库被当成能改。
5. 每次新会话都要重新交代一遍运行环境、端口和脚本入口。

这些问题用一份短文件就能解决大半。

## 做法：写一份可执行的工作空间使用手册

第一步，把文件放在仓库根目录，并纳入版本控制。OpenClaw 在工作空间加载时会读取它。

第二步，按“机器能直接执行”的格式写，不要写散文。一个最小骨架如下：

```markdown
# AGENTS.md

## Project
One-line goal: ...

## Commands
- Install: `pnpm install --frozen-lockfile`
- Test: `pnpm test:unit`
- Build: `pnpm build`
- Lint: `pnpm lint`
- Never run `rm -rf`, `git push --force`, `git reset --hard` without explicit confirmation.

## Layout
| Path | Purpose | Read/Write |
|------|---------|------------|
| `src/` | source code | read-only |
| `scripts/` | automation scripts | read/write |
| `dist/` | build output | write only |
| `tmp/` | scratch files | write only |

## MCP / Plugins
- `filesystem`: read-only. Do not modify tracked files.
- `github`: create PR only after summarizing diff.
- `postgres`: SELECT only, use read-only connection.

## Stop Conditions
Stop and report when:
- A command fails twice.
- The task requires secrets not present in env.
- The change set exceeds 20 files.
```

第三步，把“禁止项”写清楚。模型对否定句有时会忽略，所以要放在独立段落，并用 `Never`、`Do not` 开头。

## 踩坑点

- 文件太长：超过 300-500 行后，模型容易只读开头或遗漏中间规则。建议拆成 `AGENTS.md` 加 `docs/agent/` 下的细分文件，主文件只放指针和硬约束。
- 只写“适当”“可能”“建议”：这些词无法约束行为。改成“必须使用”“禁止执行”“优先选择”。
- 把 secret 写进 AGENTS.md：即使仓库私有也不要写 token、密码、内网地址。用环境变量引用。
- 把 AGENTS.md 当 TODO：任务列表会频繁变化，放进契约文件会稀释真正的规则。动态任务应放在会话里或单独 issue。
- 忘记更新：目录变更、命令改名后不同步 AGENTS.md，AI 会照着旧规则执行。

## 可复用建议

- 用表格描述目录和 MCP 权限，模型对结构化信息的遵循度更高。
- 每个规则用命令式短句，每条只表达一个约束。
- 把“停止条件”写进去，避免 AI 在失败后反复重试或扩大操作范围。
- 定期审计 AGENTS.md：删掉不再使用的命令，合并重复规则，保持它能在 2 分钟内读完。
- 如果 OpenClaw 支持，可以将 AGENTS.md 与项目级 memory 或 context 配置配合使用，但不要把两者职责混在一起：AGENTS.md 管工作空间边界，memory 管跨会话偏好。

## 总结

AGENTS.md 不是魔法，也不是另一种 README。它是一份简短、明确、可被 AI 稳定读取的工作空间使用手册。写好它，能减少大量重复说明和误操作；写坏它，只会增加上下文噪音。对 OpenClaw 的自动化、MCP 和插件场景来说，越早把规则固化下来，后期越省事。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/2992e284c0fa7b99.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/60888f5e1b473552.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/3a25191ee45f2b01.png)

