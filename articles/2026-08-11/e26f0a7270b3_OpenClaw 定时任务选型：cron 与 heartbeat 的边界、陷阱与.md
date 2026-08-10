---
title: OpenClaw 定时任务选型：cron 与 heartbeat 的边界、陷阱与工程化实践
feedId: 32485
source: 综合讨论
publishedAt: 2026-08-11
---

定时任务几乎是每个 Agent 或自动化管线都会碰到的需求——周期性采集数据、定期巡检、按时触发下游通知。OpenClaw 在插件和 Agent 定义中内置了两种定时驱动方式：**cron** 和 **heartbeat**。两种方式表面相似，但在调度语义、容错行为和适用场景上有明显区别。选错不仅会造成时序混乱，还可能引入隐蔽的并发重叠或漏执行。

本文基于真实工程踩坑经历，把两者的差异、用法和避坑点讲清楚。

---

## 一、背景：两种定时机制的差异

在 OpenClaw 的配置模型里，一个 Agent 或插件可以声明 `schedules` 列表，每个条目指定 `type` 为 `cron` 或 `heartbeat`。

- **cron**：以 Unix cron 表达式定义触发时间点，例如 `0 8 * * 1-5` 表示工作日早 8:00。调度器会按指定时间点发起一次任务执行。它依赖系统时钟，语义是“在时间到达 X 时执行”。
- **heartbeat**：以固定时间间隔执行，例如 `interval: 60s`。它不关心绝对时间点，只是在上一次执行完成后，等待固定间隔再启动下一次。语义是“每过 Y 秒执行一次”。

从实现上来看，cron 调度由 OpenClaw 内嵌的 cron 库驱动（基于系统时区），heartbeat 则是 Agent 运行时的 ticker 循环。这导致它们对进程重启、任务耗时过长等情况的反应截然不同。

---

## 二、典型用法与配置步骤

**场景 1：业务报表每 6 小时生成一次**

适合使用 cron，因为它严格按时钟边界触发，便于对账和可追溯性。

配置示例：
```yaml
# agent.yaml
schedules:
  - type: cron
    spec: "0 */6 * * *"
    timezone: "Asia/Shanghai"
    task: generate_report
    timeout: 300s
```

- `spec`: cron 表达式，支持秒级扩展（6 位）。
- `timezone`: 时区设置，避免容器默认 UTC 导致本地时间偏差。
- `timeout`: 任务最大执行时间，超时会被 context 取消。

**场景 2：实时监控 websocket 连接心跳**

heartbeat 更合适，因为它只需要固定间隔检查一次状态，不受绝对时间影响。

```yaml
schedules:
  - type: heartbeat
    interval: 15s
    task: check_ws_connection
    timeout: 5s
```

heartbeat 的任务是“触发后直接执行”，执行完成后开始等待 `interval`，再触发下一轮。若任务执行本身耗时 2 秒，实际循环周期约为 17 秒。这一点在计时敏感的场景需要仔细评估。

---

## 三、踩坑集锦

### 1. cron 重叠执行

**问题**：若 cron 任务执行时间超过间隔周期，默认调度器会再次触发，导致同一任务并发运行。例如每 5 分钟执行一次的数据抽取，由于源库慢查询跑了 7 分钟，第 6 分钟又一次触发，造成两个实例同时写目标表。

**解决**：OpenClaw 提供了 `singleton` 选项，设为 true 后，如果上一个实例还在运行，新的触发将被忽略。
```yaml
schedules:
  - type: cron
    spec: "*/5 * * * *"
    task: slow_etl
    singleton: true
    timeout: 600s
```
结合 `timeout` 防止死等。

### 2. heartbeat 的累积偏移

heartbeat 以“任务结束 + interval”计算下一次触发。如果任务偶发耗时波动，实际间隔会抖动。对于严格周期性的采集（如每秒采样一次传感器数据），这种累积偏移会导致采样时间戳不均匀。

**改进思路**：需要精确等间隔时，可以在任务内部记录绝对开始时间，用 `time.Ticker` 的修正逻辑自行调度，或者改用 cron 的秒级表达式（如 `@every 1s`）以获得更接近固定时点的触发，不过要额外注意性能开销。

### 3. 时区陷阱

cron 的 `timezone` 若不显式设置，会随容器环境变量取值，可能导致预想北京时间却在 UTC 触发。heartbeat 无此时区问题，因为没有绝对时间点。

**建议**：所有 cron 调度都显式指定 `timezone`，且与任务日志中的时间戳一致。

### 4. 进程重启后的行为差异

- cron：基于时间点，进程重启后，若启动时间已过触发点，不会再补执行（默认策略）。如果有强制补跑需求，需在任务侧实现启动检查。
- heartbeat：进程启动后立即开启 ticker，所以会从“现在”开始按间隔执行，因此可能改变原本的运行相位。

对于需要高可靠性的周期性任务，应当考虑将调度与执行分离，例如由外部调度器（Argo Workflows、Kubernetes CronJob）触发 OpenClaw 提供的 HTTP endpoint，这样责任更清晰。OpenClaw 内置的 heartbeat 更适合长期存活的 Agent 守护式任务。

---

## 四、可复用的工程建议

| 需求特征 | 推荐方式 | 关键配置 |
|---------|--------|---------|
| 按固定时间点执行（如每天 2:00）| cron | 指定 timezone，设置 singleton: true |
| 固定间隔循环，不重绝对时间 | heartbeat | 评估执行耗时，小心累积偏移 |
| 任务可能长于间隔 | cron 或 heartbeat 均有风险 | 使用 singleton，加分布式锁 |
| 多实例部署，任务只需单次执行 | 两者配合外部锁 | 引入 Redis/etcd 锁，只在获得锁时执行 |

另外，无论哪种调度，都建议为任务设置 **超时 context** 并在任务内部检查 `ctx.Done()`，避免僵尸 goroutine 泄漏。OpenClaw 的 `timeout` 字段会注入 context，任务实现方应积极响应。

---

## 五、总结

cron 和 heartbeat 并不是“谁比谁好”的问题，而是语义不同。cron 管理的是**时刻**，heartbeat 管理的是**间隔**。选择前先问自己：这个任务关心的是每天都在 8:00 执行，还是每 30 分钟跑一次？答案决定了你应该站在哪一边。

在真实复杂场景下，往往还需要组合使用：用 heartbeat 维持 Agent 心跳和健康检查，用 cron 触发业务逻辑，再用外部锁保证单实例。把调度和执行解耦，是提高系统抗脆性的长效解法。

---

