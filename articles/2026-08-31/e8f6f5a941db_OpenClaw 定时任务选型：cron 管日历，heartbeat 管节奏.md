---
title: OpenClaw 定时任务选型：cron 管日历，heartbeat 管节奏
feedId: 35512
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景

OpenClaw 里做自动化时，定时场景通常分成两类：一类是“每周一早上发摘要”“工作日 9 点到 18 点检查工单”，关心的是**日历时间**；另一类是“每 30 秒看看有没有新消息”“每 5 分钟做一次心跳保活”，关心的是**固定节奏**。OpenClaw 中常见两种触发方式：cron 表达式按日历时间触发，heartbeat 按固定间隔触发。很多人把它们混用，结果不是任务堆积，就是重启后行为不符合预期。

## 问题

一个典型故障：用户想让 agent 每 5 分钟拉取一次 MCP 数据源，用了 cron `*/5 * * * *`。表面没问题，但容器重启后赶上第 4 分钟，第一次触发只间隔 1 分钟就执行；如果服务在整点短暂不可用，这一轮就漏掉。另一个故障：用户用 heartbeat 每 30 秒执行一次“营业时间内的提醒”，结果半夜也在空跑，还因为任务执行超过 30 秒造成并发重入。

本质不是谁更好，而是 **cron 解决“什么时候做”，heartbeat 解决“多久做一次”**。两者在重启补偿、时区、重入控制上的语义不同。

## 做法/步骤

1. 先画出任务目的：是日历事件还是节奏事件。
2. 日历事件用 cron，并显式设置时区：

```yaml
automation:
  - name: morning_brief
    schedule: "0 9 * * 1-5"
    timezone: Asia/Shanghai
    action: run_plugin:briefing
```

3. 节奏事件用 heartbeat，设置 interval、timeout 和重入保护：

```yaml
heartbeat:
  - name: inbox_watch
    interval: 30s
    timeout: 20s
    skip_if_running: true
    jitter: 2s
    action: mcp:check_inbox
```

4. 明确启动行为：heartbeat 可配置 initial_delay，避免冷启动风暴；cron 可配置是否补跑，按业务决定漏一次还是重复一次更安全。
5. 给任务加幂等键。拉取类任务用游标或时间戳，不重复入库；发送类任务加 dedupe key。

> 示例仅表达语义，具体字段以你当前 OpenClaw 版本的配置校验为准。

## 踩坑点

- **时区陷阱**：cron 默认跟随进程时区，容器常是 UTC。务必显式 `timezone`，否则国内“早上 9 点”可能实际在 17 点触发。
- **heartbeat 重入**：interval 小于执行时长时，默认可能并发。必须设 `timeout < interval`，并开启 `skip_if_running` 或分布式锁。
- **重启补偿**：cron 默认不一定补跑错过的点；heartbeat 重启后会立即开始下一轮。对需要“不漏”的任务，用 heartbeat + 状态检查更合适；对“只在准点发生”的任务，接受漏一次比重复更安全。
- **jitter 缺失**：多实例或多任务同时 heartbeat 会形成同频共振，给上游 MCP/API 造成突发压力。加 1–3 秒 jitter。
- **空跑成本**：heartbeat 周期过短会放大 token 和工具调用成本。能用事件驱动或长轮询的场景，不要用短 heartbeat 硬扫。

## 可复用建议

默认规则：**日历语义用 cron，节奏/健康/兜底用 heartbeat**。如果任务需要“错过也能补”，优先 heartbeat；如果任务必须“只在指定时间点”，用 cron。所有定时任务都配 timeout、skip_if_running、timezone，并记录 last_run、last_result。上线前用 dry-run 和模拟时间做一次 24 小时快进，观察午夜、跨时区、重启后行为。

## 总结

OpenClaw 的 cron 是日历触发器，适合报告、定时发布、营业时间任务；heartbeat 是节拍器，适合监控、轮询、重试和状态对齐。选型时先问一句：任务是由时间点定义，还是由节奏定义？把时区、超时、重入、补跑四个参数配清楚，比纠结哪个更高级更实际。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/a4de21e7691ec0c5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/b1d64469e84387bc.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/0012ed3124e2c89c.png)

