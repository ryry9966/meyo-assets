---
title: OpenClaw 定时任务选型：cron 与 heartbeat 怎么区分
feedId: 35506
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景

在 OpenClaw 的 Agent 编排与自动化实践里，定时任务通常有两种来源：一种是“到某个时间点执行”，比如每天 9 点拉数据、每周生成报表；另一种是“每隔固定间隔执行”，比如每 30 秒探活、每 5 分钟清理队列。对应到 OpenClaw 的调度机制，就是 cron 和 heartbeat。两者看起来都是定时触发，但语义完全不同，选错会带来重复执行、时间漂移和假死。

## 问题

最常见的错误是把 heartbeat 当 cron 用。例如需要“每天 9 点发日报”，却配置 `interval: 24h`。这会让执行时间跟随进程启动时间，一旦重启或上次回调阻塞，时间就会整体偏移，离 9 点越来越远。反过来，只是需要“每 30 秒探测一次队列是否卡住”，却用 cron 表达式 `*/30 * * * * *`，虽然也能跑，但会强制秒级对齐，并给调度器增加不必要的精确时间负担，还不容易处理执行耗时带来的误差。

关键不是哪个更高级，而是任务面向“日历时间”还是“运行节奏”。

## 做法：按任务性质选择

### 1. 判断时间锚点

有明确自然时间点的任务选 cron。比如“工作日 9:00”“每小时第 10 分钟”“每周一 3:00”。cron 表达的是绝对时间语义，适合和外部世界对齐。  
如果任务只关心“从上一次结束后再过多久”，没有精确时刻要求，选 heartbeat。它表达的是相对时间语义，适合内部节奏。

### 2. 看是否依赖时区/日历

cron 可以配置时区，能自然处理“北京时间 9 点”。heartbeat 不关心时区，只关心过去了多久。因此涉及跨时区、冬夏令时或“每天一次”的需求，优先 cron。

### 3. 判断任务是否可能重叠

cron 任务如果执行时间超过间隔，可能并发跑多个实例，需要设置防重叠；heartbeat 回调如果做重活，会阻塞下一次心跳。所以 heartbeat 回调必须极短小，只做检查/上报。

一个参考配置思路（示例，具体以你的 OpenClaw 版本为准）：

```yaml
# 日历时间任务：每天 9 点执行
triggers:
  - name: daily_report
    type: cron
    expression: "0 9 * * *"
    timezone: Asia/Shanghai
    max_instances: 1
    task: run_report

# 运行节奏任务：每 30 秒探活
heartbeat:
  interval: 30s
  timeout: 5s
  task: health_check
```

## 踩坑点

1. **cron 时区不对**：容器/服务器默认 UTC，导致任务在错误时间跑。务必显式设置时区，并在上线前打一条测试触发记录。
2. **cron 任务重叠**：上次没跑完，下一次又到点，可能同时跑多个实例。对策是设置 `max_instances: 1` 或分布式锁，并保证任务幂等。
3. **heartbeat 回调阻塞**：在回调里直接调 MCP、读数据库或做网络请求，耗时超过 interval，下一次心跳被推迟，间隔逐渐漂移。正确做法是把耗时操作丢到异步队列，heartbeat 只做状态采集和上报。
4. **heartbeat 假死**：进程活着、定时器在跑，但业务逻辑已经卡死。heartbeat 照常发出，外部看到“心跳正常”，实际服务不可用。探活最好携带业务指标，如队列长度、上次成功处理时间戳，而不是只返回“我还活着”。
5. **用 heartbeat 模拟定点任务**：`interval: 24h` 没有日历语义，重启、阻塞、手动触发都会造成偏移。精确任务必须回到 cron。

## 可复用建议

- 绝对时间/日历语义选 cron，相对节奏/探活选 heartbeat。
- cron 任务必须幂等，并设置防重叠。
- heartbeat 回调保持轻量，不执行重业务。
- 监控两类任务的状态：cron 看是否按时触发，heartbeat 看间隔是否稳定。
- 关键业务调度用 cron，存活检查用 heartbeat，必要时组合：cron 触发工作流，heartbeat 监听执行状态。

## 总结

OpenClaw 中的 cron 和 heartbeat 回答的是两个不同问题：cron 回答“什么时间执行”，heartbeat 回答“每隔多久执行/是否还活着”。把面向日历的批量任务交给 cron，把面向节奏的探活与轻量周期动作交给 heartbeat，再各自补齐防重叠、轻回调、业务指标检查，定时任务才不会变成定时事故。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/6fc885eae4e2b2a3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/69dd764ec98f6dbe.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/0addd12467739cfc.png)

