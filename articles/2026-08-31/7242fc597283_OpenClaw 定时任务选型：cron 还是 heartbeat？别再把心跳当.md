---
title: OpenClaw 定时任务选型：cron 还是 heartbeat？别再把心跳当闹钟
feedId: 35527
source: 综合讨论
publishedAt: 2026-08-31
---

# OpenClaw 定时任务选型：cron 还是 heartbeat？别再把心跳当闹钟

在 OpenClaw 里挂定时任务时，很多人习惯把所有周期逻辑都塞进 cron；另一部分人则习惯了 heartbeat 的“每隔 N 秒跑一次”，把关键调度也放在 heartbeat 里。结果要么任务偶发延迟，要么日志被心跳刷爆。本文基于实际踩坑，聊聊两者边界。

## 背景：两个不同的时间模型

OpenClaw 的 cron 与 heartbeat 本质上是两种时间驱动模型。cron 是“日历时间”：到点触发，比如每天 02:00、每 5 分钟一次，由调度器统一计算下一次执行时间。heartbeat 是“间隔时间”：服务启动后，每隔固定时长触发一次，通常用于内部健康检查、状态同步、会话保活。

在多数配置中，cron 写的是标准五段/六段表达式，heartbeat 写的是 interval 秒数。它们都能执行 Agent 动作，但语义完全不同。

## 问题：混用带来的副作用

我见过几个典型问题：

- 用 heartbeat 执行日报生成，间隔 60 秒，结果服务重启后触发时间漂移，报表时早时晚。
- 用 cron 做健康检查，每分钟一次，节点有时还没起来，错过就错过了，下次要等一分钟。
- 把清理任务同时挂在 cron 和 heartbeat 上，导致重复执行甚至并发冲突。
- heartbeat 任务执行时间超过 interval，造成任务堆叠，内存持续上涨。

## 做法/步骤：一个简单选型流程

**步骤一：列出所有周期任务，并标记两个属性：**

1. 是否需要“准点执行”？
2. 是否有副作用（写库、发消息、调外部 API）？

**步骤二：按下面矩阵分类：**

- 准点执行 + 有副作用 → cron
- 准点执行 + 无副作用 → cron 或 heartbeat 均可，但建议 cron
- 不准点 + 有副作用 → heartbeat（但需加幂等和防重入）
- 不准点 + 无副作用 → heartbeat

**步骤三：写配置。**

示例（OpenClaw 配置片段，格式以当前版本为准）：

```yaml
scheduled_tasks:
  - name: daily_report
    type: cron
    cron: "0 2 * * *"
    action: generate_report
    timezone: Asia/Shanghai

  - name: session_heartbeat
    type: heartbeat
    interval: 30s
    action: sync_sessions
    jitter: 5s
```

**步骤四：验证。**

cron 任务要验证时区、表达式、错过补偿；heartbeat 要验证执行时长是否稳定小于 interval，最好加上 jitter 避免所有实例同时触发。

## 踩坑点

1. **heartbeat 不会“补跑”**：服务停机期间的心跳不会在恢复后补发，这与 cron 的 catch-up 策略不同。不要用 heartbeat 做需要保证次数的逻辑。
2. **cron 的时区问题**：容器内默认 UTC，配置 cron 时最好显式指定 timezone，否则会在错误的时间执行。
3. **heartbeat 重叠**：如果任务耗时可能大于 interval，必须加锁或限制并发，否则多个 tick 同时跑。
4. **cron 表达式误区**：`*/5 * * * *` 是“每 5 分钟”，不是“从启动后每 5 分钟”。如果服务 10:03 启动，第一次触发仍是 10:05，而不是 10:08。这种“边界对齐”行为可能不符合直觉。
5. **日志刷屏**：heartbeat 默认会打印 INFO 日志，间隔短时日志量很大。建议对 heartbeat 单独设置日志级别。

## 可复用建议

- 一句话判断：**需要“到点执行”用 cron；需要“活着就做”用 heartbeat。**
- 所有 heartbeat 任务都应写成幂等，且最好无外部副作用；如果必须有副作用，加分布式锁。
- cron 任务要监控“最后一次成功执行时间”，而不只是“最后触发时间”，否则静默失败不容易发现。
- 如果任务既需要准点又需要保活，可以拆成两个：cron 执行主逻辑，heartbeat 只做轻量状态更新。
- 在 OpenClaw 中，cron 适合跑批量、报表、清理、外部数据拉取；heartbeat 适合做会话保活、缓存刷新、节点心跳上报。

## 总结

cron 和 heartbeat 不是“哪个更好”的问题，而是“时间语义”是否匹配。把 heartbeat 当闹钟用，会让调度失去确定性；把 cron 当心跳用，则可能错过关键保活窗口。先分清任务的时间属性，再选择机制，能避免大部分定时任务坑。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/8dcb0b7382bd483b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/5c221beb46608d9c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/b1db1c427fc52b6e.png)

