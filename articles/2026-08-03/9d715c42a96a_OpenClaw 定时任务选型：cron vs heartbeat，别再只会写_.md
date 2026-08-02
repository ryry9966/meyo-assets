---
title: OpenClaw 定时任务选型：cron vs heartbeat，别再只会写*/5了
feedId: 31414
source: 综合讨论
publishedAt: 2026-08-03
---

# OpenClaw 定时任务选型：cron vs heartbeat，别再只会写 */5 了

## 背景：一个容易被忽视的触发器设计

在 OpenClaw 的插件和自动化编排里，定时触发是最常见的需求之一。几乎每份“每日总结”“数据同步”“健康检查”的 workflow 都会用到。

OpenClaw 提供了两种原生的定时触发器：**cron** 和 **heartbeat**。但多数用户的配置文件中，永远只会出现 cron，heartbeat 几乎成了“房间里的大象”——没人提，更没人用。直到有一天你需要做一个“当上游服务心跳丢失时立刻降级”的规则，才发现 cron 并不合适，甚至会在高频检测场景下把调度器打爆。

这篇帖子从工程实践的角度，把两种触发器的适用边界、配置细节和坑点一次性讲清楚。

## 问题：为什么“只有 cron”是不够的

cron 的核心模型是 **基于绝对时间点** 触发，比如每天 9:00、每 5 分钟一次。但很多场景其实不关心“现在几点”，而是关心：

- 从上次执行到现在，是否满足某个条件？
- 某个事件停止发生多久后，才需要我介入？
- 我要保持一个“守护循环”，而不是抢占固定时间片。

典型的例子：监控上游 API 的心跳、检测队列积压、保持 WebSocket 连接的活性、周期性刷新 token。这些场景如果硬用 cron，要么频率太高浪费资源，要么时延不可控。

heartbeat 正好是为这类需求设计的。它的模型是 **固定间隔的迭代循环**，不受时钟对齐影响，天然适合“每隔 X 秒执行一次守护逻辑”。

## 做法/步骤：两种触发器的配置与代码结构

### 1. cron 配置示例

在 OpenClaw 的 `orchestrator.yaml` 中定义：

```yaml
triggers:
  - name: daily-report
    type: cron
    spec: "0 9 * * *"
    workflow: generate-daily-report
```

代码层需要返回 `ExecuteResult`，调度器根据退出状态决定是否视为成功，并记录下次触发时间。关键点：**cron 任务必须是幂等的**，因为存在重复触发的可能（后文详述）。

### 2. heartbeat 配置示例

heartbeat 的结构完全不同，它不依赖时间表达式：

```yaml
triggers:
  - name: connection-watchdog
    type: heartbeat
    interval: 30s
    max_iterations: 0   # 0 表示无限循环
    workflow: check-upstream-health
```

对应的 workflow 插件内部通常是一个循环体，每次执行后会等待 `interval`，然后再次触发。heartbeat 任务需要自行维护状态（例如连续失败次数），不适合做无状态的短任务。

### 3. 混合编排：cron 启动 + heartbeat 终止

实用的模式是用 cron 在固定时间拉起一个长期运行的 heartbeat，到点再由 cron 或 condition 终止。例如交易时段才需要的行情守卫：

```yaml
triggers:
  - name: start-guardian
    type: cron
    spec: "30 8 * * 1-5"
    workflow: start-heartbeat-guardian
  - name: stop-guardian
    type: cron
    spec: "0 17 * * 1-5"
    workflow: stop-heartbeat-guardian
```

## 踩坑点：这些都是真实线上爆炸过的

### 坑 1：cron 的时区不是容器时区

OpenClaw 调度器内部使用 UTC 解释 cron 表达式，而容器的时区可能是 CST。直接写 `0 9 * * *` 会在 UTC 9:00 执行，而不是北京时间 9:00。解决方案是显式配置时区：

```yaml
triggers:
  - name: morning-task
    type: cron
    spec: "0 1 * * *"
    timezone: "Asia/Shanghai"
```

很多“任务怎么调都不准时”的问题，最后发现是时区没对齐。

### 坑 2：cron 重叠执行导致资源耗尽

如果上一次 workflow 还没结束，下一次 cron 触发又来了，默认行为是 **再启一个新的执行实例**。在高负载或慢任务场景下，这会快速撑大线程池，最终导致调度器整体假死。

必须通过 `concurrency_policy` 控制：

```yaml
concurrency_policy: Forbid  # 或 Replace
```

`Forbid` 直接跳过本次触发，`Replace` 会终止旧实例启新实例（需任务支持优雅终止）。没有这个设置，生产环境迟早出事。

### 坑 3：heartbeat 的间隔不是“精确”的

heartbeat 的 `interval` 是 **任务结束到下一次开始** 的等待时间，而不是两次开始的间隔。如果你的任务本身执行了 3 秒，interval 为 5 秒，那么实际触发间隔是 3+5=8 秒。如果对频率有硬性要求，需要在任务内部做时间补偿，或者用 `execution_timeout` 强行限制单次执行时长。

### 坑 4：heartbeat 任务必须是无尽循环

结束条件是 `max_iterations` 或手动停止触发器，不是在 workflow 代码里写个 `return` 就行。如果你在 heartbeat 流程中直接退出，调度器会视作任务异常终止，可能反复重试。正确的停止方式是调用 `stop_trigger` API。

## 可复用建议：一张决策表就够了

在实际选型时，下面这张表比文档更好用：

| 特征 | 选 cron | 选 heartbeat |
|------|---------|--------------|
| 触发依据 | 绝对时间 | 固定延迟/循环 |
| 时区敏感度 | 高 | 无 |
| 任务性质 | 短平快的批处理 | 有状态的守护逻辑 |
| 并发风险 | 需显式控制 | 天然单实例 |
| 资源模型 | 突发抢占 | 稳定长驻 |
| 典型场景 | 报表、数据同步 | 心跳检查、连接保洁、衰减探测 |

如果仍然难以决定，可以遵循一条简单原则：**任务执行时间 < 1分钟且无需维护历史状态，用 cron；否则先考虑 heartbeat。**

另外，所有 cron 任务都建议加一个监控，统计“最近一次执行成功时间”，超过两倍周期没更新就告警——这是防止静默失败的最低成本方式。

## 总结

OpenClaw 的 cron 和 heartbeat 不是“差不多”的替代品，它们是两种完全不同的调度模型。cron 管“什么时候做”，heartbeat 管“一直做，直到条件变化”。大多数自动化项目中，真正脆弱的往往是那些全天候守护链路，而你需要的不是更短的 cron 周期，而是把合适的任务交给 heartbeat。

下次写 `*/5` 之前，花一分钟问问自己：这到底是一个“闹钟”，还是一个“心跳”？

---

*本文基于 OpenClaw 0.9.x 调度器实践，配置示例以真实工程为原型，已脱敏。*

---

