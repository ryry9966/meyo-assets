---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 35364
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景

在 OpenClaw 里编排 Agent 或插件时，定时触发是常见需求。官方提供两类调度原语：`cron` 和 `heartbeat`。它们看起来都能“定时跑”，但底层语义不同：`cron` 绑定墙上时钟，适合“每天 9 点执行”；`heartbeat` 绑定相对间隔，适合“每 30 秒检查一次”。

实际项目里，很多任务跑不稳，不是因为业务逻辑错误，而是调度类型选错：用 `heartbeat` 做日级对账，结果重启后触发时间漂移；用 `cron` 做秒级探活，结果表达式难维护且容易重叠。本文从工程实践角度整理选择方法和避坑点。

## 问题

两类任务的核心区别是时间锚点：

- `cron`：绝对时间，例如 `0 9 * * *` 表示每天 09:00（默认 UTC）。
- `heartbeat`：相对间隔，例如 `interval: 30s` 表示上次触发后 30 秒再触发。

选择时先问一个问题：这个任务是否需要对齐自然时间？

需要对齐：日报、周报、定时同步、窗口期任务 → `cron`。
不需要对齐：健康检查、队列消费、缓存刷新、心跳上报 → `heartbeat`。

## 做法/步骤

1. 定义任务时先写清触发语义，不要直接写表达式。例如“每个工作日 08:30 拉取上游数据”应选 `cron`；“每 15 秒检查任务队列深度”应选 `heartbeat`。

2. 在 OpenClaw 配置中分离调度参数和业务逻辑。一个最小示例：

```yaml
tasks:
  - name: daily_report
    type: cron
    schedule: "30 8 * * 1-5"
    timezone: "Asia/Shanghai"
    timeout: 120s
    overlap: skip
  - name: queue_watcher
    type: heartbeat
    interval: 15s
    jitter: 3s
    timeout: 10s
    overlap: skip
```

3. 给任务加幂等键。`cron` 任务可以用 `YYYYMMDD_HHMM` 作为批次号；`heartbeat` 任务可以用业务对象 ID + 触发窗口，避免重复执行造成副作用。

4. 对 `heartbeat` 设置 `jitter`。如果是多实例部署，所有实例同一时刻唤醒会对下游造成冲击。`jitter` 把触发时间随机分散，能显著降低峰值。

5. 对 `cron` 设置 `catch_up` 策略。错过调度时，是立即补跑一次还是跳过，需要在配置中明确。对账类任务一般选择补跑；通知类任务建议跳过。

6. 监控每次执行：开始时间、结束时间、状态、队列深度、重叠次数。OpenClaw 的任务执行记录可以导出到现有监控系统，重点关注 `overlap` 和 `timeout` 两类指标。

## 踩坑点

- **时区不一致**：`cron` 默认 UTC，如果服务器时区或团队认知是本地时间，会出现“配置写 8 点，实际 16 点跑”的问题。务必显式设置 `timezone`。

- **任务耗时超过间隔**：`cron` 间隔太短或 `heartbeat` `interval` 小于实际执行时长时，如果不设 `overlap: skip`，会堆积多个并发实例。常见于网络超时未设置的情况。

- **heartbeat 漂移**：如果实现为“执行完后 sleep interval”，实际间隔等于 `执行时长 + interval`。正确做法是按固定频率调度，或记录上次触发时间，差值补偿。OpenClaw 的 `heartbeat` 通常按固定间隔触发，但如果自定义 wrapper 要注意。

- **分布式重复触发**：多个 OpenClaw 实例同时运行同一 `heartbeat` 任务时，如果没有分布式锁或选主，会重复执行。对幂等任务影响小，对计费、推送等任务必须加锁。

- **夏令时/闰秒**：`cron` 在夏令时切换日可能重复或跳过一小时内的任务。如果业务对时间窗口敏感，要评估是否需要 `cron` 之外的补偿机制。

- **heartbeat 停止恢复**：暂停后恢复，可能会立即触发一次“补跑”，如果业务有状态（如“距上次心跳超过 5 分钟则告警”），需要处理边界，否则会误报。

## 可复用建议

1. **先分类再配置**：把任务分为“定时批处理”和“常驻巡检”两类。前者用 `cron`，后者用 `heartbeat`。不要因为某个任务“需要定时跑”就默认 `cron`。

2. **业务逻辑保持幂等**：无论哪种调度，都要假设任务可能被重复触发。外部 API 调用要设置超时、重试上限和结果校验。

3. **调度与业务解耦**：OpenClaw 只负责触发，业务结果写入状态存储（如 KV 或消息队列）。这样即使调度异常，也能快速定位是“没触发”还是“触发了但业务失败”。

4. **默认加 jitter**：对 `heartbeat` 尤其重要。哪怕单实例，给 1-5 秒的 jitter 也能避免固定节律被下游识别为机器人流量或造成共振。

5. **设置 overlap 策略**：`skip` 比 `allow` 更安全。除非任务本身设计为可并发，否则不要允许重叠执行。

6. **记录调度元数据**：每次执行记录 `trigger_type`、`scheduled_at`、`fired_at`、`finished_at`。这些字段在排查时会比日志更有用。

## 总结

`cron` 和 `heartbeat` 不是“哪个更高级”，而是两种不同的时间语义。选错调度类型，往往会在运行一段时间后才暴露问题。实践中的判断顺序是：先确定时间锚点，再选择调度原语，然后用 `timezone`、`jitter`、`overlap`、幂等键和监控补齐稳定性。把这些基础做对，比引入更复杂的编排逻辑更能减少线上事故。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/a7c19e9328a5c81f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/56199835614fe170.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/984ecdb673113e5f.png)

