---
title: OpenClaw 定时任务选型：cron 与 heartbeat 的边界不是“间隔长短”
feedId: 35518
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景

在 OpenClaw 里做自动化，定时任务几乎是最常见的基础设施。无论是周期调用 MCP 工具、驱动插件逻辑，还是让 Agent 按固定节奏检查外部系统，都会遇到两种原语：`cron` 和 `heartbeat`。

不少实践者第一反应是按“cron 是精确定时，heartbeat 是固定间隔循环”来区分，但真正影响选择的不是间隔长短，而是**时间语义、错过容忍度、执行阻塞模型**。如果只看间隔，很容易把本该用 cron 的任务写成 heartbeat，或者反过来，导致任务在错误的时间触发、重复执行或静默丢失。

## 问题

实际开发中经常出现三类模糊场景：

1. “每 5 分钟跑一次”到底算 cron 还是 heartbeat？
2. cron 任务错过了执行窗口，会不会自动补跑？
3. heartbeat 上一个 tick 没结束，下一个 tick 又来了怎么办？

这些问题如果不先厘清，配置写得越多，后期排障成本越高。

## 做法 / 步骤

### 1. 先判断任务性质：墙钟驱动 vs 节奏驱动

- **cron** 解决的是“到点做”：比如每天 09:00 发日报、每周一 00:30 生成周报、每月 1 号清理归档。它绑定日历时间，错过就是错过，系统通常不会无条件补跑。
- **heartbeat** 解决的是“持续做”：比如每 30 秒检查一次任务队列、每 5 分钟同步一次外部状态、每 10 秒采集一次指标。它不关心“现在几点”，只关心“是否到了下一个 tick”。

简单决策路径：

```
是否必须在某个绝对时间点触发？
├─ 是 → 用 cron
└─ 否 → 是否要固定频率持续运行、允许轻微漂移？
         ├─ 是 → 用 heartbeat
         └─ 否 → 可能根本不需要定时任务
```

### 2. cron 配置示例（以 OpenClaw 当前配置结构为准，不同版本字段可能微调）

```yaml
scheduled_tasks:
  - name: daily_report
    type: cron
    schedule: "0 9 * * *"
    timezone: "Asia/Shanghai"
    action: run_daily_report
    catch_up: false
    timeout_seconds: 300
```

关键点：

- 显式设置 `timezone`，否则容器默认 UTC 会让任务在凌晨执行。
- `catch_up: false` 表示错过窗口后不补跑，这对日报类任务通常是合理的。
- 设置 `timeout_seconds`，避免任务卡死占住调度线程。

### 3. heartbeat 配置示例

```yaml
scheduled_tasks:
  - name: queue_check
    type: heartbeat
    interval: 30s
    action: check_queue
    timeout_seconds: 10
    on_overlap: skip
```

关键点：

- `interval` 不是 cron 的替代品，它只关心“间隔”。
- `on_overlap: skip` 表示上一个 tick 未结束时，直接跳过本次，而不是排队等待。这样可以避免重入。
- heartbeat 内部不要做重活。如果某个 tick 需要长时间运行，应拆到外部队列或异步任务里，否则 tick 会持续漂移。

## 踩坑点

1. **cron 时区不一致**：OpenClaw 运行在容器里，系统时区常为 UTC。业务按东八区写 `0 9 * * *`，实际会在北京时间 17:00 触发。务必在配置里显式声明时区。

2. **cron 错过窗口不报错**：如果任务在 09:00 因调度器重启或资源不足没执行，且 `catch_up` 为 false，它会静默跳过。排查时日志没有异常，但用户发现日报没发。建议对“错过”事件打日志或监控。

3. **heartbeat 重入导致资源泄漏**：默认情况下，如果上一次 tick 还没结束，下一次 tick 可能继续启动，导致并发访问同一资源、临时文件堆积或数据库连接耗尽。配置 `on_overlap: skip` 或内部加互斥锁。

4. **heartbeat 长期运行后状态累积**：每 tick 创建新连接、临时目录或缓存对象，项目跑几天后内存和句柄上涨。需要为 heartbeat 任务设计清理逻辑，确保每个 tick 结束即释放资源。

5. **多实例并发调度**：如果 OpenClaw 部署了多个副本，cron 和 heartbeat 都可能被重复触发。尤其是 heartbeat，多副本同时 tick 会引发竞态。需要在任务内部使用分布式锁或把调度集中到单实例。

## 可复用建议

- **用“墙钟敏感度”作为第一判断标准**：任务是否必须在某个具体时间发生？是则 cron，否则 heartbeat。
- **给所有定时任务加幂等键**：例如用 `YYYY-MM-DD` 或 batch ID 标记每次执行，即使重复触发也不会产生副作用。
- **为 heartbeat 设置最大运行时长**：超过 `timeout_seconds` 强制中断，避免 tick 线程被永久占用。
- **监控调度漂移**：记录计划执行时间与实际执行时间的差，超过阈值告警。heartbeat 漂移通常意味着任务过重或系统负载过高。
- **定期审计任务列表**：删除不再需要的 heartbeat，避免“幽灵任务”持续消耗资源。

## 总结

cron 和 heartbeat 在 OpenClaw 中不是一个“高级/低级”或“精确/模糊”的二元选择。它们的核心区别在于：cron 面向墙钟时间，失败模型是“错过即跳过”；heartbeat 面向固定节奏，失败模型是“漂移与重入”。选择时先问自己两个问题：这个任务是否必须在某个绝对时间点发生？它是否允许错过一次而不影响业务？

工程上两者完全可以共存，但边界要清晰。cron 负责“到点做”，heartbeat 负责“持续做”，不要让 heartbeat 去伪装 cron，也不要让 cron 去承担高频循环。把时间语义、补跑策略、重叠处理、资源生命周期都写进配置和文档里，才能避免定时任务从“省事”变成“定时炸弹”。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/0387a1bc6548396b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/ddbaf7315aa43e90.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/ce92804e56eb8954.png)

