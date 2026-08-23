---
title: OpenClaw 定时任务选型：cron 与 heartbeat 的边界在哪里？
feedId: 34301
source: 综合讨论
publishedAt: 2026-08-23
---

## 背景

在 OpenClaw 里做自动化，常见两种定时触发：cron 和 heartbeat。它们都能让 Agent 或插件“定期跑”，但内核不同。cron 是日历驱动，heartbeat 是间隔驱动。选错通常不会马上报错，而是表现为漏任务、重复消费、限流或状态冲突。

## 问题

一个典型场景：我想让 OpenClaw 每 30 秒检查一次收件箱，有新邮件就处理。到底用 cron 还是 heartbeat？

如果只看“每 30 秒”，两者都能写。但行为差异很快会暴露：

- cron：每 30 秒启动一个新任务实例。如果处理耗时超过 30 秒，会叠加。
- heartbeat：更接近常驻心跳，每次 tick 先检查条件，再决定是否执行，通常可以携带上一次状态。

所以关键不是“多久一次”，而是“是否允许跳过、是否依赖上次结果、是否需要条件判断”。

## 做法

### 1. 先判断任务性质

固定时间点任务用 cron。比如每天 09:00 汇总、每周一清理、每小时拉取一次数据。这类任务不关心上一次是否成功，每次执行目的明确。

条件触发或持续探测用 heartbeat。比如监听队列、等待某个文件出现、检查服务状态，满足条件才执行动作。

### 2. cron 配置示范

```yaml
tasks:
  - name: daily-report
    type: cron
    schedule: "0 9 * * *"
    timezone: "Asia/Shanghai"
    action: run_report_workflow
    timeout: 10m
    overlap: skip
```

要点：

- `overlap: skip` 或外部锁，避免上次没跑完又启动。
- `timeout` 必须有，防止僵尸任务占用 worker。
- 时区显式声明，不要依赖系统默认。

### 3. heartbeat 配置示范

```yaml
tasks:
  - name: inbox-watch
    type: heartbeat
    interval: 30s
    condition:
      field: unread_count
      gt: 0
    action: process_inbox
    dedupe_key: last_message_id
    max_interval: 5m
```

heartbeat 更适合带 `condition`，只在条件满足时进入 action。`dedupe_key` 或游标避免重复处理同一事件。

> 不同版本配置字段可能略有差异，但思路一致，请以你当前 OpenClaw schema 为准。

## 踩坑点

1. **cron 重叠执行**  
   如果任务处理时间大于间隔，cron 会再起一个实例。必须设置 `overlap: skip` 或用 Redis 锁。否则定时任务越多，越容易把 worker 打满。

2. **heartbeat 过密导致限流**  
   30 秒一次还好，但有些人用 1 秒一次去轮询 API，很容易触发限流。建议从 30s 起步，必要时用指数退避或 `max_interval` 拉长。

3. **时区不一致**  
   cron 表达式默认可能按 UTC 解析。日志显示跑了，但时间不对。统一在配置里写 `timezone: "Asia/Shanghai"`。

4. **heartbeat 任务不是“永远在线”**  
   OpenClaw 重启或 worker 切换时，heartbeat 的本地内存状态会丢。不要把关键游标只放内存，应该落到文件、KV 或消息队列。

5. **两者混用时的状态竞争**  
   如果同一份数据既被 cron 处理，又被 heartbeat 处理，很容易出现双写。建议一个资源只有一个触发源；如果必须混用，加上 `dedupe_key` 或版本号。

## 可复用建议

- 固定日历周期 → cron  
- 事件/条件驱动 → heartbeat  
- 需要保证不漏触发 → cron + 幂等  
- 需要实时性但能容忍短暂延迟 → heartbeat + 条件判断  
- 长任务必须设 `timeout` 和 `overlap` 策略  
- heartbeat 间隔不要低于 10s，除非你非常清楚下游能承受

一个简单决策表：

| 特征 | cron | heartbeat |
| --- | --- | --- |
| 触发依据 | 日历时间 | 固定间隔 + 条件 |
| 适用 | 日报、清理、批量拉取 | 监听、探测、状态机 |
| 重叠风险 | 高，需要锁 | 中，需要去重 |
| 状态依赖 | 弱 | 强 |
| 典型间隔 | 分钟~天 | 10s~分钟 |

## 总结

cron 和 heartbeat 不是替代关系。cron 解决“什么时间做”，heartbeat 解决“条件满足就做”。在 OpenClaw 中，优先用 cron 处理确定性任务，用 heartbeat 处理外部事件或状态探测。真正决定稳定性的不是选哪个，而是你有没有处理重叠、去重、超时和时区。把这两件事分开，自动化任务会少很多莫名其妙的故障。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/2fed1ac27dcc1577.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/dca7ecf164707d42.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/342fc4dbaaeb0543.png)

