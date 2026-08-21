---
title: OpenClaw 定时任务选型：cron 按点跑，heartbeat 按节奏跑
feedId: 34107
source: 综合讨论
publishedAt: 2026-08-22
---

# OpenClaw 的 cron vs heartbeat：两种定时任务怎么选

## 背景

在 OpenClaw 里做自动化，经常需要让 Agent 定时执行：早上发摘要、每 10 分钟查一次邮件、每天清理会话。配置里常见两类调度：cron 和 heartbeat。很多实践者一开始都按“哪个好写”来选，结果要么任务漏跑，要么重复跑，要么把 API 额度打满。

## 问题

cron 和 heartbeat 的触发逻辑完全不同：

- cron：按日历时间点触发，比如每天 09:00。
- heartbeat：按固定间隔触发，比如每 300 秒一次，不关心现在是几点。

选错的典型后果：

- 用 heartbeat 做“每天早上 9 点日报”，服务重启后间隔起点漂移，报告时间越来越偏。
- 用 cron 做“每 5 分钟轮询消息”，分钟级表达式没写对时会在边界附近高频触发，或漏掉关键窗口。
- 两类任务没有幂等保护，同一个任务重叠执行，导致重复发消息或重复写数据。

## 做法/步骤

### 1. 先判断任务性质

问两个问题：

- 这个任务和时间点强相关吗？比如“工作日 9:30 前发站会摘要”。是 → cron。
- 这个任务需要持续、均匀地检查吗？比如“每 10 分钟检查 MCP 连接是否掉线”。是 → heartbeat。

### 2. cron 配置示例

以 OpenClaw 常见的 scheduler 配置片段为例，字段可能因版本略有差异：

```yaml
scheduler:
  - name: morning-digest
    type: cron
    expression: "0 9 * * 1-5"
    timezone: Asia/Shanghai
    task:
      agent: digest-agent
      prompt: "生成昨日未回复消息摘要"
    retry:
      maxAttempts: 2
      backoffSeconds: 60
```

关键点：不要只写 `0 9 * * *`。容器默认时区通常是 UTC，会导致北京时间 17 点才跑。显式设置 `timezone`。

### 3. heartbeat 配置示例

```yaml
scheduler:
  - name: mailbox-watch
    type: heartbeat
    intervalSeconds: 600
    jitterSeconds: 15
    task:
      agent: mail-agent
      prompt: "检查新邮件，只处理未读且重要度高的邮件"
    concurrency: 1
```

`concurrency: 1` 很重要：如果上一次检查还没结束，下一次触发不会新起任务，避免堆积。`jitterSeconds` 是随机抖动，避免多实例同时醒来打爆上游。

### 4. 执行保护

无论选哪种，至少要加三件事：

- 幂等：任务开头检查 `task_id + date/interval` 是否已执行，避免重复。
- 超时：给 Agent 调用设置 `timeoutSeconds`，防止一个任务挂住整个调度循环。
- 可观测：每次执行写结构化日志，至少包含触发类型、触发时间、任务名、结果、耗时、是否跳过。

## 踩坑点

### cron 时区是重灾区

OpenClaw 跑在容器里时，系统时区常为 UTC。配置写 `0 9 * * *` 以为是早上 9 点，实际是北京时间 17 点。解决：显式配置时区，并在日志里打印触发时间。

### cron 错过不补跑

如果服务在触发时刻刚好重启或网络抖动，cron 默认不会补跑。需要确认你的调度器是否支持 `catchUp` 或 `missedPolicy`。对“日报”类任务，宁可在启动时做一次补偿检查，也不要假设它一定按时跑过。

### heartbeat 间隔过短

heartbeat 很容易被当成“实时监听”。如果设成每 10 秒跑一次 Agent，API 调用和上下文开销会迅速上升。建议：

- 轮询类任务最短间隔不低于 60 秒，除非上游明确支持高频。
- 用增量查询和游标，不要每次全量拉取。

### 任务重叠

heartbeat 不会等上一次完成。如果没有 `concurrency: 1` 或分布式锁，前一个任务还在处理，后一个又起来，可能出现重复回复。解决：任务开始前抢锁，拿不到锁就跳过；或者设置 `concurrency: 1`。

### 把重任务直接塞进 scheduler

scheduler 只是触发器，不应该承载大量业务逻辑。重任务应由 scheduler 唤起的 Agent/子流程执行，并考虑队列。否则一个慢任务会阻塞后续触发。

## 可复用建议

- **日报、定时抓取、定时清理**：用 cron，显式时区，做错过补偿。
- **MCP 连接健康检查、队列轮询、心跳上报**：用 heartbeat，设置合理间隔和抖动。
- **通用规则**：
  - cron 表达式加注释或命名，别只留裸表达式。
  - heartbeat 任务必须可幂等重入。
  - 所有定时任务都记录 `trigger` 字段，方便区分是 cron 还是 heartbeat。
  - 对上游 API 做限流预算，把 heartbeat 间隔和每次调用成本一起评估。
  - 定期检查调度日志，不要等到漏跑三天才发现。

## 总结

cron 解决“什么时候跑”，heartbeat 解决“每隔多久跑”。在 OpenClaw 里，优先按时间语义选型：日历型用 cron，节奏型用 heartbeat。选对之后，把时区、幂等、超时、抖动和日志补齐，定时任务才能真正稳定运行，而不是变成新的运维债。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/c7740941f1561303.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/1606ae387ce565c2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/3bc1e5da1405b4d2.png)

