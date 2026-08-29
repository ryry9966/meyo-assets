---
title: OpenClaw 定时任务选型：cron 和 heartbeat 的边界与实践
feedId: 35250
source: 综合讨论
publishedAt: 2026-08-29
---

在 OpenClaw 里做定时自动化，常会同时看到 cron 和 heartbeat 两种触发器。很多人第一反应是“heartbeat 就是更细粒度的 cron”，然后把所有周期任务都改成 heartbeat；或者反过来，用 5 段 cron 去模拟秒级巡检。结果往往是任务重叠、触发时间漂移，或者日志里一片噪音。

## 背景与问题

OpenClaw 里的 cron 和 heartbeat 解决的是两类不同问题：

- **cron**：按“墙钟时间”触发，适合“每周一 9 点”“每小时第 0 分”“工作日 18:30”这类日历型任务。
- **heartbeat**：按“相对节奏”触发，适合“每 30 秒看一次队列积压”“每 5 分钟同步一次状态”“连续 3 个心跳失败就告警”这类状态型任务。

如果选错，典型表现是用 heartbeat 做每日 9:00 汇总，重启后触发时间会从启动时间开始偏移，永远不在 9:00 整跑；用 cron 做 30 秒一次健康检查，表达式精度不够，还容易出现上一次没跑完下一次又启动。

## 做法/步骤

### 1. 先判断任务属于“日历型”还是“状态型”

- 日历型：需要“某个固定时刻做一次”，选 cron。
- 状态型：需要“每隔一段时间观察一次”，选 heartbeat。

一个简单判断：如果任务说明里能写出明确时间点，例如“每天 10:00 发报告”，就是 cron；如果只能写出间隔，例如“每 20 秒查一次队列”，就是 heartbeat。

### 2. cron 配置示例

```yaml
trigger:
  type: cron
  expr: "0 9 * * 1-5"   # 工作日 09:00；若调度器为 6 段，改为 0 0 9 * * 1-5
  timezone: Asia/Shanghai
action: daily_report
```

### 3. heartbeat 配置示例

```yaml
trigger:
  type: heartbeat
  interval: 20s
  jitter: 3s          # 避免多实例同时唤醒
  timeout: 15s
  overlap: skip       # 上次没跑完则跳过本次
action: queue_check
```

### 4. 防重叠

cron 任务必须做运行锁，可以用 Redis、文件锁或数据库唯一 run_id。heartbeat 任务用 `overlap: skip`，或者自己记录 running 状态。对于执行时间不稳定的任务，heartbeat 间隔必须大于 p95 执行时长，否则会快速堆积。

## 踩坑点

- **cron 位数写错**：从 crontab 粘来的 5 段表达式，在 6 段解析器里会被当成秒级字段，触发时间完全偏移。上线前先 dry-run 看下未来 10 次执行时间。
- **heartbeat 太短导致叠跑**：20 秒间隔的任务如果执行 25 秒，没有 overlap 控制会同时跑多个实例，消耗 token、连接或直接把下游打满。
- **把 heartbeat 当 cron 用**：重启后心跳从启动时间开始计算，不保证在整点或固定秒触发。需要整点任务就显式用 cron。
- **时区问题**：容器内 UTC 与本地时区不一致，cron 会在错误时间触发。务必在 trigger 里写 timezone 并验证。
- **日志噪声**：高频 heartbeat 每个 cycle 打 info 日志会淹没错误，建议成功不打日志，失败做聚合。

## 可复用建议

- 决策规则：有明确“几点做”用 cron；有明确“每隔多久检查/同步”用 heartbeat。
- 给 heartbeat 加 jitter，防止多实例或重启后同时唤醒给下游造成压力。
- 所有定时任务都应幂等，因为 cron 和 heartbeat 都可能因锁失效或重试重复执行。
- 为每个定时任务记录 `last_success_at` 和 `last_error`，排障时先看“上次成功时间”，而不是只看“任务有没有被触发”。
- 对延迟敏感但不需要固定时刻的任务，优先 heartbeat；对时间敏感、需要人为可读排程的任务，cron 更合适。

## 总结

cron 解决“在正确的时间做一次”，heartbeat 解决“以正确的节奏持续观察”。选择前先确认任务属性，而不是看间隔精度。工程上，两者都要做防重叠、幂等和可观测。把 cron 用于日历触发，把 heartbeat 用于状态闭环，基本能避开大多数定时任务的坑。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/4650be0dcb936666.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/b4af852316500a73.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/64157296c43bfe8b.png)

