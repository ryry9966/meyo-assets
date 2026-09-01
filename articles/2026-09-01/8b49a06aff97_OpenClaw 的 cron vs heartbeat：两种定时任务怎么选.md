---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 35659
source: 综合讨论
publishedAt: 2026-09-01
---

## 背景

OpenClaw 里做自动化时，绕不开两类定时逻辑：一类是“到点就做”，比如工作日早上 9 点生成报告；另一类是“每隔一会儿就检查/推进一次”，比如每 30 秒扫描过期会话、每 60 秒探活一次 MCP server。

这两类需求分别对应 OpenClaw 中常见的 `cron` 和 `heartbeat` 两种定时机制。它们看起来都能触发任务，但如果混用，容易出现重复执行、任务堆叠、错过触发窗口，或者 heartbeat 把 agent 拖进 busy loop。

这篇不讨论“哪个更强”，只从工程化角度说清楚：什么任务用 cron，什么任务用 heartbeat，以及怎么配、怎么避坑。

## 问题

实际使用中，最容易出现三个问题：

1. **该用 cron 的地方用了 heartbeat**：比如每小时拉一次外部数据，用 heartbeat 每 60 秒检查一次，再在 handler 里判断是否到整点。逻辑能跑，但白白增加调度次数，也容易因为判断条件写错导致漏触发。
2. **该用 heartbeat 的地方用了 cron**：比如“会话过期后自动回收”。如果只配一个 cron 在每小时第 0 分钟执行，那么 10:59 过期、11:00 被清理还算能接受；但如果 11:01 过期，就要等到 12:00 才处理。状态收敛太慢。
3. **两种机制叠加但没做幂等**：同一个 handler 既被 cron 触发，也被 heartbeat 触发，结果同一个动作执行两次。

## 做法/步骤

### 1. 先把任务按“时间语义”分类

| 任务类型 | 更合适 | 原因 |
| --- | --- | --- |
| 每个工作日 9 点生成报告 | cron | 绝对时间触发 |
| 每 10 分钟同步一次外部数据 | cron 或 heartbeat 皆可 | 若要求整点对齐用 cron |
| 每 30 秒扫描过期会话 | heartbeat | 相对周期、状态收敛 |
| MCP server 健康检查 | heartbeat | 需要持续探测 |
| 失败任务重试队列 | heartbeat | 周期性检查并推进状态机 |

一条简单判断：**任务是“按表触发”还是“按状态驱动”**。前者偏 cron，后者偏 heartbeat。

### 2. 配置上尽量明确区分

以常见配置样式为例：

```yaml
tasks:
  daily_report:
    type: cron
    expr: "0 9 * * 1-5"
    timezone: Asia/Shanghai
    action: generate_daily_report

  session_gc:
    type: heartbeat
    interval_sec: 30
    max_runtime_sec: 20
    action: collect_expired_sessions
```

`cron` 任务要显式指定 `timezone`，不要依赖系统默认时区。`heartbeat` 任务要给 `max_runtime_sec` 或等价限制，避免上一个 tick 还没结束，下一个 tick 又进来。

### 3. handler 入口做幂等和防重

无论哪种定时机制，handler 都应该是幂等的。常见做法是用一个轻量锁：

```python
def generate_daily_report():
    if not acquire_lock("daily_report", ttl=300):
        return
    try:
        # 实际业务
        run_report()
    finally:
        release_lock("daily_report")
```

对于 cron，锁是防重复触发；对于 heartbeat，锁是防止上一个执行体还没退出，下一轮又启动同一个任务。

### 4. 记录执行状态

至少记录 `last_start_at`、`last_success_at`、`failure_count`。heartbeat 因为运行频率高，不能只看是否启动，还要看“最后一次成功是什么时候”。如果某个 heartbeat 任务连续失败 10 次，说明 handler 内部逻辑或依赖的 MCP 工具已经不可用，而不是调度器的问题。

## 踩坑点

- **cron 时区不一致**：容器内是 UTC，配置却是 `Asia/Shanghai` 表达式，结果任务比预期早/晚 8 小时。排查时先看 `date`，再看调度器实际触发的本地时间。
- **cron 错过触发窗口**：如果 OpenClaw 当时不在线，恢复后是否补跑很关键。日报类任务通常“补跑”价值低，可以直接跳到下一次；但补数据类任务可能必须在恢复后补一次。如果没有原生 `misfire_policy`，就自己在 handler 里根据 `last_success_at` 判断是否需要补做。
- **heartbeat interval 太小**：比如每 2 秒跑一次 MCP 工具调用，但工具响应 5 秒。这不会让任务更实时，反而会堆积队列、占用连接。一般 heartbeat 间隔应大于最坏情况下单次任务耗时的 2~3 倍。
- **heartbeat 当成精确计时器**：heartbeat 的语义是“检查并处理”，不是“每 N 秒必须执行一次业务”。它可能因为调度延迟、agent 忙碌、GC 停顿而偏移。需要严格周期时，应该用 cron。
- **状态判断依赖本地内存**：heartbeat 里保存“上次处理到哪”时，如果放在进程内变量，重启后状态丢失。需要持久化到文件或数据库。

## 可复用建议

1. **cron 负责“到点开跑”，heartbeat 负责“周期收敛”**。不要把整点判断塞进 heartbeat，也不要用 cron 硬模拟高频轮询。
2. **给所有定时 handler 加锁和超时**。cron 防重，heartbeat 防堆叠。
3. **MCP 调用必须带 timeout 和 retry**。定时任务里调 MCP 工具，不能因为一次工具卡死拖垮整个 agent。
4. **监控 last_success_at 而不是 last_start_at**。启动成功不代表业务成功，heartbeat 尤其容易“跑得欢但没有实际产出”。
5. **一个任务只挂一种触发源**。除非 heartbeat 只是健康检查，否则不要让同一业务逻辑同时被 cron 和 heartbeat 驱动。

## 总结

OpenClaw 里 cron 和 heartbeat 不是竞争关系，而是两种不同的调度语义。cron 适合绝对时间表，heartbeat 适合周期性状态检查。选择的关键不是任务大小，而是任务的时间语义和失败容忍度。

配置时先分类，再加锁，最后看 `last_success_at`。做到这三点，大部分定时任务的问题都可以在变成线上事故之前被发现。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/b1b773438eff98d6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/811f8aee69cb183b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/f79bed279d3e726e.png)

