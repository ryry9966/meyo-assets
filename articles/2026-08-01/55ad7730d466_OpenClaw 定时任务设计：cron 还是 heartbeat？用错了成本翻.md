---
title: OpenClaw 定时任务设计：cron 还是 heartbeat？用错了成本翻倍
feedId: 31146
source: 综合讨论
publishedAt: 2026-08-01
---

## 背景：两种“定时”的本质不同

在 OpenClaw 里驱动 Agent 或自动化流程，离不开时间触发器。你最常见到的两种方式是：

- **cron 触发器**：基于表达式的定时调度，到点就执行。
- **heartbeat 触发器**：以固定间隔轮询某个条件，条件满足才真正执行业务逻辑。

表面看它们都带“时间间隔”，但工程上差异巨大。cron 是“时间驱动”，heartbeat 是“状态驱动”。混淆使用的代价包括资源浪费、任务堆积甚至漏触发。

## 问题：什么时候该用哪个？

举个例子：你有一个 MCP 服务，需要每收集到 100 条日志就打一次分析请求。如果直接用 cron 每 10 分钟跑一次，可能日志只有 20 条却白白调用分析 API；如果改成 heartbeat 每 30 秒检查队列长度，达到阈值才触发，能节省大量无效执行。

另一个反面场景：每天零点生成报表。用 heartbeat 每秒检查一次时间是否到零点，完全是暴殄资源，cron 一句 `0 0 * * *` 就能解决。

简单原则：
- 任务执行时间点固定（比如每天、每小时整点） → cron
- 任务需要根据系统状态动态决定是否执行 → heartbeat

但现实往往没这么纯粹，理解两者的运行时行为和 OpenClaw 的实现细节，才能做对决策。

## 做法：在 OpenClaw 中配置和使用

### cron 任务示例（数据同步）

在 OpenClaw 的 pipeline 定义中，添加 cron 触发器：

```yaml
triggers:
  - type: cron
    expression: "0 */6 * * *"   # 每6小时执行一次
    timezone: "Asia/Shanghai"
    action:
      invoke: sync_data_pipeline
```

OpenClaw 调度器根据 cron 表达式触发，内部管理任务队列。注意时区必须显式指定，否则默认 UTC 会导致“半夜跑任务”的错觉。

### heartbeat 任务示例（队列监控）

同一个 pipeline，也可以定义 heartbeat 触发器：

```yaml
triggers:
  - type: heartbeat
    interval: 30s
    condition:
      metric: mcp.msg_queue.length
      operator: gte
      value: 100
    timeout: 5m
    action:
      invoke: analyze_log_batch
```

heartbeat 实际是 OpenClaw 的事件循环内定期评估条件，条件成立即触发。设置 `timeout` 是防止条件持续成立时重复触发——比如一次处理完 100 条日志后，队列长度仍然≥100，可能造成风暴。合理做法是让业务 action 返回 `trigger_reset` 或手动重置计数器。

## 踩坑点：不注意这些，上线就背锅

### 1. cron 任务堆叠
如果 pipeline 执行时间超过 cron 间隔，OpenClaw 默认不会跳过新触发；若没有并发控制，多个实例会同时跑，数据可能重复或写冲突。
**解法**：设置 `concurrency: 1` 并启用 `skip_if_running`，保证单实例顺序执行。如果你的场景需要并行，必须保证幂等。

### 2. heartbeat 的“心跳麻痹”
heartbeat 轮询间隔太短，CPU 和 DB 查询压力陡增；轮询间隔太长，事件响应延迟不可控。监控队列时，30 秒可能错过突发流量。
**解法**：根据业务可接受的延迟倒推 interval，并监控条件评估的耗时。可配合指数退避：平时 30s 轮询，当队列长度接近阈值时缩短到 5s。

### 3. 时区陷阱
cron 表达式的时区在配置文件和运行环境可能不一致，导致实际触发时间偏移若干小时。heartbeat 虽然无此问题，但日志时间戳若带时区需统一处理。
**解法**：全局使用 UTC 存储和计算，只在展示层转换。OpenClaw 的 `timezone` 字段务必填写。

### 4. 幂等缺位
任何定时任务都可能重入——无论是 cron 重复调度还是 heartbeat 条件瞬间抖动。没有幂等设计，数据脏了排查极其痛苦。
**解法**：业务 action 入口加幂等键（如 `task_id + 日期窗口`），或利用数据库唯一约束拒绝重复写入。

## 可复用建议

1. **决策树先行**：在画流程图时就区分时间驱动与状态驱动，防止技术选型错误。
2. **混合使用**：cron 做周期性重建，heartbeat 做事件触发。例如 cron 每小时全量同步一次，heartbeat 实时监控增量队列补漏。
3. **监控告警**：为 cron 和 heartbeat 分别设置超时、失败率、延迟监控。OpenClaw 的 execution log 能直接推送到你的可观测系统。
4. **测试边界**：心跳条件等于阈值时、跨天时、管道重启时，这些 edge case 必须在 CI 里用时间模拟覆盖。
5. **资源配额**：heartbeat 高频率轮询可能消耗大量协程或连接，记得限制全局并发触发数。

## 总结

cron 和 heartbeat 不是“哪种更好”，而是“哪种更匹配你的执行模型”。cron 擅长周期性、稳定的工作负载，成本低、心智负担轻；heartbeat 则把触发权交给实际状态，适合事件驱动的灵敏场景。在 OpenClaw 里，它们可以同时存在、互相补位，关键是在 pipeline 定义时把并发、超时、条件和幂等四件事一次性想清楚。工程上的稳妥，从来都比概念上的炫酷重要。

---

