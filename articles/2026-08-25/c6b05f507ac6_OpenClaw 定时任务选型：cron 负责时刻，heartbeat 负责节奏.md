---
title: OpenClaw 定时任务选型：cron 负责时刻，heartbeat 负责节奏
feedId: 34651
source: 综合讨论
publishedAt: 2026-08-25
---

# OpenClaw 定时任务选型：cron 负责时刻，heartbeat 负责节奏

## 背景
在 OpenClaw 里挂 Agent、MCP 工具或插件时，很多场景需要周期性执行：定时拉取数据、巡检资源、清理会话、发送汇总、保活长连接。OpenClaw 的调度通常提供两种入口：cron 表达式和 heartbeat 间隔。表面看都是定时器，但调度语义不同。

## 问题
cron 锚定日历时间，比如每天 09:00 执行；heartbeat 锚定运行节奏，比如进程活着时每 60 秒执行一次。如果只按“间隔大小”选型，容易遇到几类问题：

- 每天 08:00 的报告被 heartbeat 写成 08:03、08:08，时间漂移；
- 每 5 分钟巡检用 cron 配成 `*/5 * * * *`，虽然可用，但错过窗口后不会自动补，还得处理分钟对齐；
- heartbeat 任务执行时间超过 interval，导致上一轮还没结束下一轮又进来；
- 容器时区导致 cron 差 8 小时。

一句话：cron 回答“什么时候做”，heartbeat 回答“每隔多久做”。

## 做法/步骤

### 1. 先分类任务
判断任务是否需要对齐墙钟：

- 需要对齐外部系统、人类作息、账单日、整点切片的，用 cron。
- 只需要进程内保活、队列消费、缓存刷新、健康检查的，用 heartbeat。

### 2. cron 配置示例
以 OpenClaw 任务配置思路为例，具体字段以你部署版本为准：

```yaml
tasks:
  - name: morning_report
    schedule: "0 8 * * *"
    timezone: "Asia/Shanghai"
    overlap: skip
```

统一使用 timezone 字段，不要把系统时区当默认。任务内有时间判断时，也用同一时区。

### 3. heartbeat 配置示例

```yaml
tasks:
  - name: queue_drain
    interval: 60s
    overlap: skip
    max_runtime: 50s
```

heartbeat 适合“只要进程活着就持续做”的逻辑。interval 应明显大于任务平均执行时间，同时配置 overlap 策略，避免堆叠。

### 4. 验证与上线

- 联调时把 cron 改成未来 2 分钟，heartbeat interval 缩到 10s，观察触发次数和结束时间。
- 检查日志里开始/结束是否成对出现。
- 上线后看三类指标：执行次数、执行耗时、失败次数。不要只看“触发成功”。

## 踩坑点

1. **时区**：cron 在容器里默认 UTC，`0 8 * * *` 可能不是北京时间早上 8 点。显式配置 timezone，日志里同时打 UTC 和本地时间。
2. **重入**：heartbeat interval 60s，任务跑 90s，如果不配置 overlap=skip 或分布式锁，会出现并发实例。共享资源时可能重复消费。
3. **错过窗口**：cron 任务在进程冷启动或维护期可能错过。关键任务要配置 catch-up/missed 策略，或在启动时跑一次补偿。
4. **相位漂移**：heartbeat 计时器在服务重启后会重置，不能依赖它做“每天 00:00 精确截断”。这种场景必须用 cron。
5. **多实例惊群**：多副本同时 heartbeat，相同 interval 会同时唤醒。可加 jitter 或只在 leader 节点执行。
6. **幂等不足**：无论哪种调度，任务本身要有幂等键、唯一约束或执行时间窗口，否则调度重试会产生脏数据。

## 可复用建议

- **选型规则**：外部系统/人类时间用 cron；内部循环/保活/消费用 heartbeat。
- **配置模板化**：cron 必填 timezone；heartbeat 必填 interval、overlap、max_runtime。
- **统一 UTC**：存储和调度全链路用 UTC，展示层转本地时区。
- **加 jitter**：多副本或任务量较大时，给 heartbeat 加 0-5s 随机偏移，避免瞬时压力。
- **监控执行结果而非触发**：任务是否完成、耗时多少、失败原因，比“触发了几次”更重要。
- **间隔小于 1 分钟**：不要硬用 cron，很多实现最小粒度是分钟，用 heartbeat 或常驻消费者。

## 总结

OpenClaw 的 cron 和 heartbeat 不是高级/低级之分，而是两种调度语义。cron 适合对齐墙钟的定时任务，heartbeat 适合跟随进程节奏的周期任务。选型时先问：这个任务需要对哪个“时刻”负责？再配置时区、重叠策略、幂等和补偿。做到这几点，两类定时任务可以稳定共存。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/e32da630d9e3e6c0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/5d98768e8fdc9e55.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/de26b09e1b8eefbd.png)

