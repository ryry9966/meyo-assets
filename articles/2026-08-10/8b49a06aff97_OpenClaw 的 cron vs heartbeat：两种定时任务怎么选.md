---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 32361
source: 综合讨论
publishedAt: 2026-08-10
---

# OpenClaw 的 cron vs heartbeat：两种定时任务怎么选

## 背景

在 OpenClaw 的 agent 与自动化管线里，定时任务几乎是标配：定时拉取外部 API、周期巡检资源状态、守护 MCP 连接、触发模型评估流水线…… OpenClaw 内在任务调度层提供了两种最常用的触发器——**cron** 与 **heartbeat**。二者都能实现“周期性执行”，但在调度模型、并发行为、适用场景上有本质区别。

很多实践者第一次配置时容易陷入“随便选一个能跑就行”的坑，结果出现任务堆积、漂移、或者资源浪费。本文将清晰拆解二者的设计边界，并给出可直接复用的选型与配置建议。

---

## 问题：为什么不能互换

- 用 **heartbeat** 做“每整点执行的报表任务”，会导致执行时间随上次耗时漂移，一整天的执行时刻越来越晚；
- 用 **cron** 做“每 10 秒发送心跳保活”，一旦某次调用阻塞，后续触发会在准点时刻并发重入，把下游打爆。

核心原因在于两种定时器的**计时起点**和**并发模型**不同。理解它们才能写稳生产级定时任务。

---

## 触发模型对比

### cron：基于时钟的时间点调度

cron 使用类 Unix 的表达式（如 `*/5 * * * *`），在**预设的固定时刻**触发任务。OpenClaw 的调度器会按表达式计算下一次点火时间，时间一到即创建一个新的执行实例。

**关键特征：**

- 触发时刻精准，不受上次任务执行时间影响；
- 若上次任务在下一个触发点还未结束，可能发生**实例重叠**；
- 适合短周期、需对齐自然时间（如 0 分、整点）的批量作业。

### heartbeat：基于间隔的循环等待

heartbeat 更像是内嵌在 agent 里的一个守护循环：执行完一次逻辑后，**等待指定间隔时间**，再执行下一次。它不关心现在是几点几分，只关心“距离上次结束过了多久”。

**关键特征：**

- 自然避免重叠，最多只有一个实例在跑；
- 实际执行频率 = 任务耗时 + 间隔，存在**漂移**；
- 适合保持长连接、轻量探活、顺序要求严格的流式处理。

---

## 配置示例

OpenClaw 中通过项目布局或 agent 清单指定触发器，以下以典型片段说明。

**cron 任务**（每天凌晨 2 点汇总日志）：

```yaml
tasks:
  - name: daily-log-summary
    trigger:
      type: cron
      expression: "0 2 * * *"
    action: run_agent
    params:
      agent_id: log-summarizer
    concurrency_policy: forbid  # 避免重叠
```

**heartbeat 任务**（每 30 秒对 MCP 服务发探活并重连）：

```yaml
tasks:
  - name: mcp-health-check
    trigger:
      type: heartbeat
      interval_seconds: 30
    action: run_agent
    params:
      agent_id: mcp-prober
    restart_on_failure: true  # 探活异常自动重启循环
```

**注意**：cron 模式下通常需要显式处理并发策略（`forbid`、`allow`、`replace` 等），而 heartbeat 天生单实例，只需关心失败后是否重启循环。

---

## 踩坑记录

1. **时区陷阱**  
   cron 表达式默认以 UTC 调度。北京时间比 UTC 快 8 小时，直接写 `0 9 * * *` 实际是UTC上午9点，对应北京时间下午5点。须在配置中声明 `timezone: "Asia/Shanghai"` 或统一用 UTC 偏移换算。

2. **heartbeat 的“间隔”不等于“频率”**  
   如果探活逻辑平均耗时 7 秒，设置 `interval_seconds: 30`，实际每轮间隔约 37 秒。对精确频率有要求的场景需要换成 cron，或把超时控制在间隔的极小比例内。

3. **cron 重叠导致雪崩**  
   一次执行阻塞超过一个周期就触发第二个实例，进而加倍资源占用。**必须设置 `concurrency_policy: forbid` 并配合 `timeout_seconds`**，否则一旦下游慢，调度器会不断创建新实例。

4. **heartbeat 的静默失败**  
   循环内抛出未捕获异常且未配置 `restart_on_failure: true` 时，heartbeat 就直接退出，不再触发。线上表现为“探活任务悄悄死了”。最佳实践是始终启用失败重开，并接入 OpenClaw 的执行日志通知。

5. **全局间隔的误用**  
   有些用户期望 heartbeat 的 `interval_seconds` 是一个“全局时钟”，结果发现两次执行之间会在宕机恢复后补偿缺失的周期数。heartbeat 从不补跑，它是简单的时间间隔等待。

---

## 可复用的选型建议

- **任务特性决定选型**：
  - 需要“每天 10：00 生成日报” → **cron**
  - 需要“不断轮询一个流式端点，处理完即等 2 秒再拉” → **heartbeat**
  - 混合场景下可以考虑 cron 触发批处理 + heartbeat 守护辅助服务。

- **工程保障清单**：
  - cron 任务必须定义 `timeout_seconds` 和 `concurrency_policy`
  - heartbeat 任务必须设置 `restart_on_failure: true`，并监控任务存活
  - 两类任务都打上标签，方便在 OpenClaw 执行面板筛选
  - 对于分布式部署，cron 任务搭配**领导者选举锁**或仅在一个 worker 上注册，避免所有实例同时触发

- **重构警告**：  
  当你发现自己在 heartbeat 里不断计算下次几点触发，或者为 cron 写大量互斥逻辑时，大概率选错了触发器。

---

## 总结

cron 与 heartbeat 并不是性能好坏的对比，而是**调度语义的分野**。cron 面向时间点，强于日历驱动的批任务；heartbeat 面向运行循环，强于持续性的守护或轮询。把语义理清、把并发与失败模式配好，就能在 OpenClaw 上跑出稳定而不浪费的定时工作流。

记住：对时间敏感选 cron，对频率敏感但容错漂移选 heartbeat，永远为任务加上超时和明确的失败策略。

---

