---
title: HEARTBEAT.md：给 Agent 装上主动脉搏
feedId: 35587
source: 综合讨论
publishedAt: 2026-09-01
---

## 背景

在 OpenClaw 的 Agent 实践中，很多自动化链路仍然是“被动式”的：用户提问、手动触发，或者用 cron 机械地跑脚本。Agent 不知道当前进度、优先级和阻塞点，状态散落在聊天记录里，重启就丢。我们需要一种轻量、可审计、可复用的机制，让 Agent 像心跳一样定期自检、主动汇报、按需行动。

这个机制就是 **HEARTBEAT.md**。

它不是新概念，而是把“主动行为”外化为文件的工程约定。核心思路：用一个文件作为 Agent 的外部状态锚点，配合调度和 MCP 工具，让 Agent 每隔一段时间读取、判断、行动、回写。

## 问题

没有心跳文件时，常见痛点：

- 状态丢失：上下文只存在会话中，进程一重启就没了。
- 任务不可见：多个自动化脚本互相不知道对方在做什么。
- 无法判断“现在该做什么”：Agent 缺少稳定的待办来源。
- 误操作风险高：没有上次动作记录，无法追溯。
- 系统提示里写“主动干活”不靠谱：模型需要外部状态，而不是空泛指令。

## 做法/步骤

### 1. 建立 HEARTBEAT.md

把它放在项目根目录或 Agent 工作区。字段建议：

```yaml
---
last_heartbeat: 2025-04-01T08:00:00Z
last_action: "sync_issues"
pending_tasks:
  - id: gh-123
    title: "update docs"
    status: pending
    created: 2025-03-31T10:00:00Z
blocked_issues: []
health_checks:
  api: ok
  db: ok
next_actions:
  - "check new GitHub issues"
needs_approval: false
---
```

保持字段精简，避免文件膨胀。时间戳统一用 ISO 8601 / UTC。

### 2. 定义心跳间隔

建议 30–60 分钟，或每天开始/结束时。根据任务紧急度和 API 成本调整。在 system prompt 中明确：

> 每轮开始前，先读取 HEARTBEAT.md；若距离 `last_heartbeat` 超过阈值，则执行心跳流程。

### 3. 用调度触发

OpenClaw 里可以用 cron 插件或外部定时任务，调用一个脚本或 MCP 工具。脚本读取 HEARTBEAT.md，把内容作为上下文推给 Agent，或直接调用 `agent.run`。**保持单入口**，避免多个调度重复触发。

### 4. 心跳四步循环

Agent 在心跳流程中做四件事：

1. **读**：读取 HEARTBEAT.md 当前内容。
2. **检**：检查 `pending_tasks`、`blocked_issues`、`next_actions`，对比上次动作。
3. **行**：执行低风险、幂等的动作，如拉取新 GitHub Issues、发提醒、更新文档。
4. **写**：更新 `last_heartbeat`、`last_action`、清理已完成任务，必要时写回 `next_actions`。

### 5. 与外部系统联动

通过 MCP 连接 GitHub Issues、日历、邮件等，把新事件写入 `pending_tasks`，让心跳发现并处理。这样 Agent 能从“等你喂任务”变成“自己找任务”。

## 踩坑点

- **心跳过频**：会击穿 API 限额，产生大量噪声。从 60 分钟起步，观察成本再调整。
- **状态文件无限增长**：每轮心跳清理已完成的 `pending_tasks`，把历史归档到 `heartbeat.log`，只保留最近 N 条。
- **并发写入冲突**：多个 Agent 实例同时心跳，可能写坏文件。简单方案：只允许主 Agent 写，其他只读；或使用文件锁。
- **时区混乱**：统一 UTC，避免本地时区导致“过期”误判。
- **自动执行风险**：涉及删除、发布、付款等操作，必须设 `needs_approval: true` 并挂起，等人工确认。
- **模型幻觉**：在 system prompt 中强调：“只能根据 HEARTBEAT.md 中明确列出的任务行动，不得凭空创造任务。”

## 可复用建议

1. **模板化**：把 HEARTBEAT.md 模板放入项目脚手架，新项目直接复制。
2. **封装成 MCP server**：提供 `heartbeat:read` / `heartbeat:write` / `heartbeat:tick`，减少模型直接操作文件出错。
3. **保留 audit log**：每个心跳记录执行摘要，便于回溯和排障。
4. **静默模式**：没有待处理任务时，仅更新时间戳和健康检查，不输出冗长汇报，节省 token。
5. **社区共享字段扩展**：比如加入 `sla`、`retry_count`、`owner`，按项目需要调整。

## 总结

HEARTBEAT.md 本质上是一个“心跳包”：让 Agent 有节奏地自检、汇报、行动。它把主动行为从模糊的 prompt 指令变成可审计的文件协议，配合 MCP 调度和明确规则，Agent 就能从被动应答变成自驱执行。

关键是：**协议简单、状态可审计、动作可回滚**。它不是万能药，但比空喊“让 AI 主动”可靠得多。先把心跳跑起来，再逐步扩展任务源和自动化范围。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/3d506a602f10327e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/6a16abc2425a9cd0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/c4f6241ed604d5a1.png)

