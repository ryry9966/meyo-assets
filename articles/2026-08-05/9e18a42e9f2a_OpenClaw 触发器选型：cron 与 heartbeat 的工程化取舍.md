---
title: OpenClaw 触发器选型：cron 与 heartbeat 的工程化取舍
feedId: 31738
source: 综合讨论
publishedAt: 2026-08-05
---

## 背景：定时任务在 Agent 编排中的角色

多智能体协作框架里，Pipeline 不仅需要事件驱动，也依赖定时唤醒。比如每日抓取竞品价格、每 5 分钟检测某接口健康状态，或每小时汇总日志推送报告。

OpenClaw 提供了两种原生定时触发器：

- **cron**：基于类 Unix cron 表达式，在预设时间点触发
- **heartbeat**：基于固定间隔的周期性检查，常用于观测某个条件是否满足

选错触发器可能导致任务漏跑、重复执行或资源空转。本文将厘清两者的设计哲学、适用场景及避坑要点。

## 两种触发器的本质差异

**cron 是时间锚点驱动**。  
“每天 9:00 执行一次”这种语义下，9:00 这个时刻是硬约束。无论系统当时负载如何、前置数据是否就绪，一到点就点火。它适合强时序要求的工作，如日报生成、离线数据同步。

**heartbeat 是状态轮询驱动**。  
“每隔 30 秒检查是否有未处理的高优告警，有则触发通知”。heartbeat 不关心绝对时间，只关心经过指定间隔后，某一组条件是否成立。它是“等待型”任务的实现基底，适合监控、兜底补偿、异步任务的状态巡检。

从系统开销看，cron 只在触发点产生一次调度，heartbeat 则持续消耗少量资源用于判断。若检查逻辑过重或间隔太短，heartbeat 可能引入额外负载。

## 在 OpenClaw 中如何配置

OpenClaw 的 Pipeline 配置中，触发器声明位于 `triggers` 字段。一个最小化 cron 示例：

```yaml
pipeline: daily_report
triggers:
  - type: cron
    spec: "0 9 * * *"
    timezone: "Asia/Shanghai"
steps:
  - run: fetch_data
  - run: generate_report
```

heartbeat 配置：

```yaml
pipeline: alert_guard
triggers:
  - type: heartbeat
    interval: 30s
    condition: "pending_alerts_count > 5"
steps:
  - run: escalate_to_oncall
```

heartbeat 的 `condition` 是可选表达式，支持简单比较和布尔运算。未设置 condition 时，每次心跳都会触发 pipeline，相当于固定频率循环。

## 场景选择矩阵

| 场景 | 推荐触发器 | 原因 |
|------|-------------|------|
| 定时报告、数据同步 | cron | 需要精确到分钟或小时的绝对时间 |
| 队列积压监控 | heartbeat | 即时性要求灵敏，绝对时间无意义 |
| 数据补漏（如凌晨重跑失败任务） | cron | 低峰期批量处理，时间点明确 |
| 守护进程健康检查 | heartbeat | 需持续探测，间隔通常 10–60 秒 |
| 外部 API 限流窗口对齐 | cron | 与对方重置时间（如 UTC 0 点）同步 |
| 长任务状态轮询 | heartbeat | 结果返回时间不确定，轮询直到完成 |

## 踩坑实录

### cron 的时区陷阱
cron 表达式的执行时间基于 runner 的时区。如果未显式声明 `timezone`，可能使用 UTC。国内生产环境中忘记配置 `Asia/Shanghai`，导致日报在凌晨 1 点触发，而不是早上 9 点。**解法**：强制写明时区，并用运维手段校验服务器时区一致性。

### cron 的错过补偿与重叠执行
当 runner 因宕机错过某一轮 cron 触发时，默认行为在不同版本下可能不一致：有的直接跳过，有的会补跑。如果 pipeline 内包含幂等性不强的步骤（如推送消息），补跑可能重复通知。**解法**：在 pipeline 入口加业务幂等键，或在调度层增加“最大错过补偿次数”限制。

### heartbeat 的条件评估开销
heartbeat 的 condition 会在每次心跳时执行。若 condition 内调用外部数据库或 HTTP 查询，间隔 5 秒就可能打出数千次额外调用，消耗连接池。**解法**：condition 应仅限于读取本地状态或轻量变量；重查询放在 step 内并用缓存。

### heartbeat 的无界循环
不加防护的 heartbeat pipeline 一旦持续满足条件，会以间隔时间不断触发，形成风暴。**解法**：引入**熔断计数器**，如“连续触发 5 次后暂停 10 分钟”，或设置 `max_fires_per_period` 限流。

## 可复用建议

1. **cron + jitter（随机抖动）**  
   有多条 cron 在同一时刻触发时，加 1–30 秒随机延迟，避免瞬时资源争抢。可通过 pipeline 起始 step 的 sleep 实现。

2. **heartbeat 的 condition 前置熔断**  
   用一个全局状态存储“上次成功处理时间”，condition 中增加 `current_time - last_handle_time > cooldown`，防止高频重复动作。

3. **组合使用**  
   一类经典模式：heartbeat 做高频条件检测，满足后将任务写入缓冲队列；cron 做批量出队处理，避免高频写库。例如告警收敛：heartbeat 发现异常，写消息入 Redis；cron 每分钟批量读取并发送聚合通知。

4. **调试与监控**  
   - 在 OpenClaw 的管理端观察 pipeline 的触发历史，标记是 cron 触发还是 heartbeat。
   - 为 heartbeat 增加“空转”指标（condition 为假时的触发次数），用于评估 interval 合理性。

## 总结

cron 与 heartbeat 并非优劣之争，而是语义与资源的最佳匹配。cron 提供确定性的时间锚点，适用于周期性离线作业；heartbeat 提供灵活的轮询驱动，是守护和补偿型任务的基石。工程实践中需关注时区、幂等、限流与组合模式，让触发器真正成为可靠的调度引擎，而非故障源头。

---

