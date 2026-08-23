---
title: Agent 的 HEARTBEAT.md：让 AI 主动做事而不是等你提问
feedId: 34427
source: 综合讨论
publishedAt: 2026-08-24
---

## 背景

多数 agent 工作流是被动的：你问一句，它答一句。一旦关掉对话，它不会主动检查服务器、生成日报或跟踪 issue。想要主动，常见做法是写 cron 脚本调用 agent，但每次调用都是无状态的，agent 不知道上次做到哪，容易重复或遗漏。

## 问题

外部调度能唤醒 agent，却无法保持上下文。如果让 agent 自己维护 todo，又会把状态写得到处都是。我们需要一个轻量的“心跳”文件，既能让 agent 每次被唤醒时快速恢复，又能约束它只做该做的事。

## 做法：给 Agent 一个 HEARTBEAT.md

### 1. 建立状态文件

在 agent 工作区创建 `HEARTBEAT.md`，用 YAML frontmatter 存放机器可读状态，正文放任务模板。示例：

```markdown
---
last_run: 2025-01-01T08:00:00Z
next_due: 2025-01-01T09:00:00Z
state: idle
lock: none
---
# tasks
- id: check-disk
  schedule: "0 * * * *"
  status: due
  tool: mcp__filesystem__read
  args: {path: "/var/log/disk.log"}
  note: "检查磁盘使用率并写入 report.md"
```

### 2. 配置外部调度

在 OpenClaw 的 scheduled trigger 或系统 cron 里，每 10 分钟触发一次 agent，提示词固定为：

> 读取 HEARTBEAT.md，只处理 status=due 且当前时间已到 next_due 的任务。执行后更新 last_run 和任务状态。不要做其他探索。

### 3. 执行与更新

Agent 读取文件，匹配到期任务，调用 MCP 工具或插件（filesystem、github、notion 等）完成任务，然后写回状态：把 status 改为 done，更新 last_run，计算下个 due 时间，并将结果追加到 `heartbeat.log` 或通过通知插件发送摘要。

## 踩坑点

- **并发写坏文件**：多个触发器可能同时唤醒 agent。加一个 `.lock` 文件，脚本里判断锁存在则跳过，或使用原子写入。
- **Agent 自由发挥**：自然语言任务容易让 agent 读错字段或做多余操作。任务模板里的工具名、参数必须明确，提示词强调“按字段执行，不要修改结构”。
- **状态膨胀**：每次执行都写结果，文件会越来越大。只保留最近一次状态和必要计数，详细日志放单独文件。
- **时区不一致**：cron 表达式和 `next_due` 必须统一时区，建议全用 UTC。
- **失败处理**：任务失败不要直接置为 done，改成 failed 并记录错误，下次心跳重试，设置最大重试次数。

## 可复用建议

- 把 HEARTBEAT.md 当成状态机，字段固定，少用自然语言。
- 任务粒度要小，一次心跳只做一件事，比如“检查一个 API 返回码”而不是“维护整个系统”。
- 提供 dry-run 参数，提示词加“如果 dry_run=true，只输出计划不执行”，方便调试。
- 关键操作加人工确认：让 agent 把待执行命令写到 `pending_approval.md` 并通知用户。
- 结合 MCP，让 agent 真正操作外部系统，而不是只生成文本。

## 总结

HEARTBEAT.md 的价值不是让 agent 变聪明，而是给它一个稳定、可恢复的“心跳”状态。外部调度负责唤醒，文件负责保持上下文，约束提示负责防止跑偏。三者配合，才能让 agent 从被动问答变成可依赖的主动巡检。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/bd82d4f455a08b4c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/92a8a1c062cba312.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/c23adca8e115772b.png)

