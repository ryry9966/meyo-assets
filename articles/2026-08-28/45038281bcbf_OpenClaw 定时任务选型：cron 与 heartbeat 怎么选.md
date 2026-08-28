---
title: OpenClaw 定时任务选型：cron 与 heartbeat 怎么选
feedId: 35064
source: 综合讨论
publishedAt: 2026-08-28
---

## 背景

OpenClaw 里做自动化时，常见两种定时触发：cron 和 heartbeat。两者都能“到点执行”，但语义不同，选错会导致任务堆积、空转或时区翻车。本文基于实际配置经验，梳理二者的边界与选择。

## 问题

一个典型场景：我需要一个 agent 每天早上 9 点抓取 RSS 生成摘要；另一个 agent 负责监控任务队列，15 秒探一次。前者明显是日历任务，后者是固定节奏。但很多配置把两者都写成 interval，或用 cron 硬凑高频轮询，导致维护困难。

## 做法/步骤

### 1. 先看语义，再选类型

- cron：按自然时间触发，适合“每天/每周/每月某点”。典型表达如 `0 0 9 * * *`（秒 分 时 日 月 周）。
- heartbeat：按固定间隔触发，适合“每 N 秒/分钟跑一次”，与自然时间无关。

OpenClaw 配置示例：

```yaml
triggers:
  - name: daily-digest
    type: cron
    expression: "0 0 9 * * *"
    timezone: Asia/Shanghai
    task: fetch_daily_news

  - name: queue-watch
    type: heartbeat
    interval: 15s
    timeout: 8s
    task: check_queue
```

配置上，cron 需要显式时区；heartbeat 需要 `interval`，建议配 `timeout`。

### 2. 决策表

| 需求 | 选择 |
| --- | --- |
| 每天 9 点、每周一 10 点 | cron |
| 每 30 秒探活、每 5 分钟轮询 | heartbeat |
| 需要避开节假日/工作日 | cron 表达式或外部日历 |
| 进程重启后仍需对齐自然时间 | cron |
| 常驻心跳、保活、队列消费 | heartbeat |

### 3. 给任务加保护

无论选哪种，任务本身要幂等。调度层只负责触发，任务实现应能重复执行不产生副作用。

## 踩坑点

1. **时区问题最隐蔽**：容器默认 UTC，cron 配了 `9 0 * * *` 会变成北京时间下午 5 点。务必在配置中写 `timezone: Asia/Shanghai`，或统一用 UTC 再转换。

2. **cron 任务重叠**：上次没跑完，下次又触发。例如每日抓取超过 24 小时，或者触发频率高于执行时长。需要加锁或设置 `skip_if_running: true`。

3. **heartbeat 漂移与堆积**：如果任务耗时接近或超过 `interval`，会出现并发叠加。比如 10 秒间隔、任务跑 12 秒，队列会越积越多。必须设置 `timeout`，并让任务支持“如果上次未完成则跳过”。

4. **heartbeat 重启后节奏变化**：进程重启会重新计时，可能从 0 开始，导致探活间隔不固定。如果对节奏敏感，改用 cron 或持久化偏移。

5. **高频 heartbeat 空转**：每 1 秒 heartbeat 会让 agent 常驻唤醒，增加资源占用。低频业务用 cron 更省。

6. **cron 表达式字段数量**：OpenClaw 可能支持 6 位（含秒），也可能兼容 5 位。配置前先确认，避免“每天 9 点”跑成“每分钟第 9 秒”。

## 可复用建议

- **任务与调度解耦**：把具体逻辑写成独立 task/工具，触发器只做调用。这样 cron 换 heartbeat 不用改业务代码。
- **记录执行元数据**：每次运行记录开始时间、耗时、结果。出现静默失败时能快速定位。
- **统一时区**：团队统一 UTC 存储，展示时转换本地时间，减少跨地域协作混乱。
- **先 dry-run 再上线**：用短间隔 heartbeat 或未来 1 分钟的 cron 验证，确认触发链路和幂等逻辑。
- **监控漂移**：对 heartbeat 类任务，监控实际执行间隔与配置间隔的偏差，设置告警阈值。

## 总结

cron 解决“在某个自然时间点执行”，heartbeat 解决“每隔固定时间执行”。OpenClaw 里没有绝对优劣，只有语义匹配。日历任务用 cron，固定节奏/保活用 heartbeat，同时做好时区、超时、幂等和监控，定时任务才不会变成定时事故。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/47fbbe4ede02e970.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/feb299a51f52fd7f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/d637b85b39922e4c.png)

