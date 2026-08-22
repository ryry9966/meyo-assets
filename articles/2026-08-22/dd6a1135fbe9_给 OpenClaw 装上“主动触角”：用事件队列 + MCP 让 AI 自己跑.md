---
title: 给 OpenClaw 装上“主动触角”：用事件队列 + MCP 让 AI 自己跑腿
feedId: 34177
source: 综合讨论
publishedAt: 2026-08-22
---

## 背景

OpenClaw 这类 Agent 框架常被当成“高级 chatbot”：消息进来，模型回复。但很多运维、监控、自动化场景真正需要的是反过来的——没有用户消息，AI 也要按事件或时间主动处理任务：证书到期前检查、备份完成后校验、监控告警聚合、定时生成项目摘要。

这种 proactive 能力不是让模型“自己想干活”，而是给它一个稳定的外部触发入口。

## 问题

在 OpenClaw 上做 proactive，难点不在模型能力，而在工程链路：

- 外部事件如何变成 OpenClaw 会话输入；
- 定时任务与 webhook 很容易重复触发；
- 用户不在场，主动任务权限不能过大；
- 失败后容易静默，没有人发现。

## 做法/步骤

我建议把 proactive 拆成四层：**触发源 → 事件队列 → OpenClaw 会话 → 可观测结果**。

### 1. 事件队列先行

不要直接在 OpenClaw 里跑 cron 循环。用一个 SQLite 表或 Redis list 存事件，字段固定：

```json
{
  "id": "evt_01",
  "trigger_type": "cert_check",
  "payload": {"domain": "example.com"},
  "dedup_key": "cert_check:example.com:2025-06-12",
  "status": "pending",
  "limit_time": "2025-06-12T23:59:59Z"
}
```

`dedup_key` 用来去重，`limit_time` 作为 TTL，过期未执行的事件直接标记为 `expired`。

### 2. 创建受限的 proactive 会话

在 OpenClaw 中单独建一个 `proactive` 会话或项目，system prompt 只让它做三件事：读取事件、按模板调用白名单工具、把结果写回。

工具权限只允许 `read_*`、`write_run_result`、`notify_*`。执行类工具如果没有 dry-run 参数，先不要在主动任务里开放。

### 3. 触发与执行

时间驱动用 cron / systemd timer；事件驱动用 webhook 接收监控或 Git 事件。它们只负责向队列写入事件，不直接执行。

执行端可以用 OpenClaw CLI 或网关 API 触发：

```bash
openclaw run --session proactive --message "$(cat event.json)"
```

如果部署的是网关形态，也可以把事件 POST 到 inbound webhook。关键是把 `dedup_key` 一起传入，并在队列层做原子锁定：

```sql
UPDATE events SET status='running', started_at=now()
WHERE id = ? AND status='pending';
```

锁定失败就跳过，避免并发或重叠执行。

### 4. 结果与告警

让 agent 执行完调用 `write_run_result`，runner 根据返回判断是否成功。失败时通过通知工具发送到指定频道，不要只依赖模型“自己记住重试”。同时记录 tool calls 日志，便于发现权限过大或误调用。

## 踩坑点

- **重复触发最常见**：cron 与 webhook 重叠、agent 执行时间超过间隔，都会造成重复。用队列锁定 + dedup_key + TTL 三层防护。
- **主动任务比对话容易越权**：用户不在场，模型一次误判就可能把测试环境做掉。默认 dry-run，写操作二次确认或完全禁止。
- **时区问题**：定时任务统一 UTC，配置里写清 `TZ=Asia/Shanghai`。
- **上下文累积**：不要用同一个长期会话跑所有事件，建议每个事件一个新会话，或跑完清理上下文，避免上一个事件影响本次决策。
- **静默失败**：增加心跳检查，例如 24 小时无成功记录就通知，否则“主动助手”会死得很安静。

## 可复用建议

- 从只读 + 报告型任务开始：证书到期检查、备份校验、依赖更新提醒、异常日志摘要。
- 仓库结构固定为 `triggers/`、`events/`、`tools/`、`runs/`，每个主动任务一个目录。
- 事件 schema 固定，至少包含 `id`、`trigger_type`、`payload`、`dedup_key`、`ttl`、`created_at`。
- 工具白名单用 MCP server 控制，proactive 会话不要继承完整工具权限。
- 定期查看 tool_call 日志，如果某个主动任务频繁调用无关工具，说明 system prompt 或工具列表需要收紧。

## 总结

proactive 不是让模型“想主动”，而是把“外部事件 → 受限 agent → 可观测结果”这条链路做成稳定工程。OpenClaw 的 MCP/插件体系适合承接这件事，但真正决定可靠性的是队列、去重、dry-run、白名单和告警。先把一两个低风险任务跑稳，比一次接很多触发源更划算。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/7d94101075289c1e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/b94528f402cb70d8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/adada4cd056257e3.png)

