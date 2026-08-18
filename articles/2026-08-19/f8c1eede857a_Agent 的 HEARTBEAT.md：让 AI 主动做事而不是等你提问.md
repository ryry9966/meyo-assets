---
title: Agent 的 HEARTBEAT.md：让 AI 主动做事而不是等你提问
feedId: 33773
source: 综合讨论
publishedAt: 2026-08-19
---

## 背景

大多数 Agent 都活在“问答模式”里：你不发消息，它就不动。即使接入了 MCP、插件和定时任务，很多自动化仍然停留在固定脚本阶段——到点执行，没有上下文，也不会根据状态调整。

但有一类工作天然适合 Agent 主动完成：定时整理未读邮件、检查 PR 状态、清理日志、维护知识库、监控本地队列。如果在 OpenClaw 里只靠 cron 调一个“无状态 prompt”，每次运行都是冷水启动。Agent 不知道上次做到哪，也不知道这次该优先做什么。

于是我开始尝试一个做法：给 Agent 一份 `HEARTBEAT.md`，把“什么时候跑、当前状态、待办、边界、日志”都放在一个可审计文件里。它不是魔法提示词，而是一个让 Agent 能周期性自我唤醒、按规则主动做事的契约文件。

## 问题

单纯定时触发 Agent，会暴露几个工程问题：

- 状态丢失：每次运行都是新的，任务中断后无法续接。
- 边界模糊：Agent 可能误删文件、重复发送通知、越权操作。
- 上下文膨胀：把完整历史塞进 prompt，token 成本和延迟都不可控。
- 没有回滚：Agent 自动改动后，出了问题很难定位。

`HEARTBEAT.md` 的思路是：把“主动”限制在一个可预测的范围里。Agent 定期读取这个文件，根据 `tasks` 和 `rules` 决定做什么，再写回状态和日志。人随时可以查看、修改、回滚。

## 做法/步骤

### 1. 定义 HEARTBEAT.md 结构

一个最小可用的模板可以这样：

```markdown
# HEARTBEAT.md

- owner: openclaw-local
- cadence: 15m
- last_run: 2025-01-18T09:15:00Z

## State
- inbox_unread: 8
- pr_open: 3
- last_cleanup: 2025-01-17

## Tasks
- [ ] summarize unread inbox into daily-brief.md
- [ ] check PR #42 CI status, if failed post summary
- [ ] clean logs older than 7d in ~/.openclaw/logs

## Rules
- 不回复任何外部消息
- 只写 ./workspace 目录
- 超过 5 分钟的任务先写 plan 再执行
- 失败最多重试 1 次

## Log
- 09:15 heartbeat ok, 3 tasks, 0 blocked
```

结构不需要复杂，关键是 `State`、`Tasks`、`Rules`、`Log` 四块稳定存在。

### 2. 设置触发机制

在 OpenClaw 里，触发方式可以有三种：

- 写一个 MCP server，暴露 `heartbeat.run` 工具，由外部 cron 调用。
- 在 OpenClaw 插件层注册定时任务，把 `HEARTBEAT.md` 路径作为上下文注入。
- 更简单地，用 systemd timer / launchd 调 CLI：`openclaw run --heartbeat ~/.openclaw/HEARTBEAT.md`。

触发只是入口，真正的决策逻辑来自文件里的 `tasks` 和 `rules`。不要把任务写死在触发命令里。

### 3. 固定执行流程

每次 heartbeat 运行，让 Agent 严格按这个顺序走：

1. 读取 `HEARTBEAT.md`。
2. 检查 `last_run` 和 `cadence`，判断是否需要执行。
3. 过滤 `Tasks` 中未完成项。
4. 逐条对照 `Rules` 判断是否允许执行。
5. 执行任务，更新 `State` 和 `Log`。
6. 写回文件，必要时标记 `blocked` 和等待条件。

### 4. 先 dry-run，再放开

首次接入，先让 Agent 只生成执行计划，不实际修改文件或发通知。确认 `Rules` 能拦住危险操作后，再开放写权限。

## 踩坑点

### 1. 并发写入导致状态丢失

如果多个 heartbeat 实例同时运行，同时写 `HEARTBEAT.md`，会互相覆盖。解决方式是单实例运行，或者写临时文件再 `rename`。文件里可以加 `pid` 或 `lock_until` 字段，避免重入。

### 2. 上下文越写越长

`State` 和 `Log` 很容易膨胀。建议保留最近 N 条日志，超过阈值就归档到 `heartbeat-archive.md`。`State` 只保留当前决策需要的最小字段，旧状态可以由 Agent 压缩成一行摘要。

### 3. 文件监听循环触发

如果 OpenClaw 里有文件监听插件，Agent 写完 `HEARTBEAT.md` 可能再次唤醒自己。建议 heartbeat 运行期间禁用自身监听，或者把这个文件放在独立目录里。

### 4. 规则太宽导致误操作

“帮我清理旧文件”这种模糊任务很危险。规则必须是明确允许列表，例如“只允许删除 `~/.openclaw/logs/` 下 7 天前的 `.log` 文件”。默认拒绝，遇到不确定操作先记录并跳过。

### 5. MCP 工具不稳定

文件系统工具超时或权限异常时，Agent 可能中断整个 run。建议每个任务加上 timeout 和降级策略：工具失败只记录 `blocked`，不影响其他任务。

### 6. 时间解析混乱

`last_run` 用 UTC ISO8601 字符串，避免本地时区歧义。Agent 在比较时间时统一转 UTC。

## 可复用建议

- 一个 `HEARTBEAT.md` 只负责一个场景，比如 `inbox-heartbeat.md`、`repo-heartbeat.md`。
- 规则尽量写成布尔条件：“仅当 `inbox_unread > 0` 才运行”，减少空转。
- 让 Agent 在 heartbeat 结束时主动汇报结果，而不是干完就消失。
- 把 `HEARTBEAT.md` 放进 git 跟踪，出问题可以回滚。
- 给 heartbeat 使用独立 API key 或最小权限账号，限制可写路径。
- 定期检查 `Log` 和 `blocked` 任务，把频繁出现的判断固化成规则。

## 总结

`HEARTBEAT.md` 的价值不是让 Agent 变得更聪明，而是给“主动行为”加上边界和记忆。它让 Agent 从每次冷启动，变成一个有状态、可审计、可回滚的定时协作者。

如果你已经在 OpenClaw 里用 MCP 或插件做自动化，可以先把一个低频任务迁移到 heartbeat 模式。跑几天 dry-run，再逐步放开写权限。主动不是失控，规则清晰才能让 Agent 真正替你做事。

---

