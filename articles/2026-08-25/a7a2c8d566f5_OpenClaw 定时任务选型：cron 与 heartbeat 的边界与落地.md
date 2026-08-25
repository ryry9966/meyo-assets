---
title: OpenClaw 定时任务选型：cron 与 heartbeat 的边界与落地
feedId: 34731
source: 综合讨论
publishedAt: 2026-08-25
---

在 OpenClaw 里做自动化，定时任务往往是最先被用到的能力。cron 和 heartbeat 都能“定期触发”，但两者调度模型并不一样：cron 表达的是“在某个时间点执行”，heartbeat 表达的是“每隔固定间隔执行一次”。混淆使用通常不会马上报错，但会导致重复执行、漏触发或任务漂移。

## 背景：两种调度原语

OpenClaw 的 cron 适合日历语义，例如每天 09:00 生成日报、每周一清理快照、每月 1 号做归档。它依赖时间表达式，触发时刻由系统时钟和时区决定。

heartbeat 适合节拍语义，例如每 30 秒扫描一次队列、每 5 分钟做一次健康检查、每 10 秒拉取一次 MCP 状态。它更像一个持续脉冲，不关心“现在是几点”，只关心“距离上一次是否已经过了 interval”。

## 问题：什么时候不该用哪个

常见错误有两类：

- 用 cron 做高频探测：`*/1 * * * *` 每 1 分钟跑一次。能跑，但 cron 的强项不是稳定节拍；系统时间跳变、时区切换、cron 表达式边界都会影响触发。若任务需要固定频率且不绑定绝对时刻，heartbeat 更直接。
- 用 heartbeat 做日历任务：设置 `interval: 24h` 想每天跑一次。实际上启动时间会漂移，部署重启、容器重建、夏令时都会让“每天 09:00”变成“每天 09:07:23”或其他时间，不可控。

## 做法：按任务性质选择并配置

### 1. 先判断任务性质

问两个问题：

- 是否必须在某个具体时刻执行？例如“工作日 09:30 发晨报”。
- 是否只要保持固定频率即可？例如“每 30 秒检查是否有待处理任务”。

前者用 cron，后者用 heartbeat。

### 2. cron 配置示例

```yaml
schedulers:
  daily_report:
    type: cron
    expr: "0 9 * * 1-5"
    timezone: Asia/Shanghai
    task: daily_report
    concurrency: forbid
    timeout: 600s
```

这里显式设置 `timezone`，避免容器默认 UTC 导致触发时间偏移。

### 3. heartbeat 配置示例

```yaml
schedulers:
  queue_watch:
    type: heartbeat
    interval: 30s
    task: drain_queue
    timeout: 10s
    max_pending: 1
    skip_missed: true
```

`max_pending: 1` 限制最多一个待执行实例，`skip_missed: true` 在系统繁忙时跳过过期节拍，避免堆积。

### 4. 执行语义与重叠保护

cron 和 heartbeat 都可能遇到“上一次还没跑完，下一次又触发了”。建议统一配置：

- `concurrency: forbid` 或分布式锁，防止同一任务重叠。
- `timeout` 设置最大执行时长，超过后中断或告警。
- 任务内部实现幂等键，推荐使用 `scheduled_time` 或 `trigger_id` 作为业务去重依据。

## 踩坑点

- **时区不一致**：cron 在容器里默认 UTC，配置了 `0 9 * * *` 实际在本地 17:00 触发。务必显式设置 `timezone`。
- **heartbeat 漂移**：如果任务耗时接近或超过 interval，实际间隔会被拉长。固定频率模式下可能堆积，固定延迟模式下会逐渐偏慢。需要设置 `timeout` 并监控间隔。
- **多实例重复触发**：cron 在多副本部署下每个实例都会触发，需用 leader election 或分布式锁；heartbeat 可用于主备心跳，但要明确谁执行任务。
- **把 heartbeat 当保活**：如果 heartbeat 任务本身只是“发一个请求”而不检查返回状态，无法发现依赖服务异常。心跳必须携带状态判断和告警路径。
- **日志缺失**：两种调度都要记录 `trigger_time/start_time/end_time/result`，否则排障只能靠猜。

## 可复用建议

- 决策表：绑定日历时刻用 cron；固定频率、连续探测、补偿扫描、队列排水用 heartbeat。
- 所有定时任务统一配置 `timeout`、`concurrency`、失败重试策略，避免默认值导致不可控。
- 上线前用空跑模式验证：cron 查看下一次触发时间，heartbeat 观察 10 个连续间隔是否稳定。
- 多副本部署时，为 cron 增加分布式锁；heartbeat 若用于健康检查，可以把结果写入状态而不是直接执行副作用。
- 关键任务加幂等键，重复触发不应产生重复业务影响。

## 总结

cron 解决“在什么时候跑”，heartbeat 解决“每隔多久跑一次”。选型时先确认任务是否绑定日历时间，再看是否需要固定频率。两者不要互相替代：日历任务用 cron，持续探测用 heartbeat。调度层保持简单，把时间语义、重叠保护、幂等和监控做好，比纠结表达式本身更重要。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/c07a98ea98e021c0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/8189fccbec70043d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/8514bcc641fe9088.png)

