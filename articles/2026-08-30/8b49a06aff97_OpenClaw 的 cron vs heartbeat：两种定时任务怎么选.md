---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 35268
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景

在 OpenClaw 里做自动化，定时任务基本绕不开。常见需求分两类：一类是“每天 9 点拉取数据”“每周一生成报告”，另一类是“每 30 秒探活一次 MCP server”“每 5 分钟刷新一次本地缓存”。前者关心日历时间，后者只关心固定节奏。OpenClaw 提供了 cron 和 heartbeat 两种触发方式，但边界不清晰时很容易用错。

## 问题

如果不区分语义，会出现几种典型故障：

- 用 heartbeat 做“每天早上 8 点推送”，实例重启后时间基准漂移，推送点越来越偏；
- 用 cron 做“每 30 秒健康检查”，cron 通常不适合秒级高频，且多实例同时触发会造成探测风暴；
- 长任务没有重叠保护，上一轮还没结束下一轮又启动，把外部 API 打崩或产生重复副作用；
- 容器内时区默认 UTC，cron 看起来“晚了 8 小时”。

## 做法/步骤

### 1. 先分类，再选触发器

写配置前问一句：这个任务关心“现在是几点”还是“距离上次过了多久”？

- 关心日历时间 → `type: cron`
- 只想按固定间隔执行 → `type: heartbeat`

示例配置：

```yaml
tasks:
  - name: daily_report
    type: cron
    schedule: "0 9 * * 1-5"
    timezone: Asia/Shanghai
    run_with_lock: true
    timeout: 120s
    retry:
      max_attempts: 2
      backoff: 30s

  - name: mcp_health_check
    type: heartbeat
    interval: 30s
    jitter: 3s
    overlap: skip
    timeout: 8s
```

### 2. 给每类任务加执行边界

不要裸跑。至少加四件事：

- **锁/overlap**：cron 用 `run_with_lock`，heartbeat 用 `overlap: skip` 或 `wait`；
- **超时**：防止外部调用挂死；
- **重试**：只对幂等操作开，非幂等操作默认不重试；
- **jitter**：多副本或整分钟集中触发时，加 1-10 秒随机偏移。

### 3. 用日志验证触发语义

上线前观察 `next_run` 和 `last_duration`：

- cron 的 `next_run` 应该固定在日历点；
- heartbeat 的 `next_run` 大致是 `last_run + interval`，不以 0 分为锚点。

## 踩坑点

**cron 时区问题**：容器默认 UTC 很常见。`0 9 * * *` 在 UTC 下就是国内 17 点。显式配置 `timezone`，并在本地用 `TZ=UTC` 跑一次 dry-run。

**多副本重复执行**：OpenClaw 如果跑多个 worker，cron 可能被每个副本同时触发。除非任务天然幂等，否则要依赖分布式锁或指定单副本执行。不要假设“只有一个实例”。

**停机期间不补跑**：进程停机期间的 cron 一般不会补跑。如果数据完整性敏感，需要单独的补数任务，而不是依赖 cron 自动 catch-up。

**heartbeat 不是墙钟**：心跳的基准是实例启动时间。服务重启后，下一轮按新基准计算。用 heartbeat 做报表、推送、限流策略时，容易在发布后看到任务整体平移。

**长任务漂移**：heartbeat 的 next 通常是“计划开始时间 + interval”，而不是“上一次结束时间 + interval”。一个 35 秒的任务配 30 秒 interval，会造成持续追赶。要么缩短任务，要么用 `overlap: skip`，要么改成“结束后再等 N 秒”的链式触发。

**cron 整点风暴**：如果大量任务都写 `0 * * * *`，整点可能瞬时集中。给非严格任务设置 `jitter`，或把任务错开 5-10 分钟。

## 可复用建议

- **业务时间表用 cron，节奏性维护用 heartbeat。** 日报、周报、营业时间数据同步用 cron；健康检查、缓存刷新、队列消费、token 保活用 heartbeat。
- **MCP/插件调用要设超时和降级。** 定时任务调用 MCP server 时，MCP 可能冷启动或丢失，不要用默认无限等待。
- **所有定时任务都应该是可重放的。** 即使触发器偶尔重复或补跑，业务侧不应产生错误数据。做不到幂等，至少记录 `task_id` 和 `trigger_time` 方便对账。
- **先在单副本环境观察 3-5 轮再放量。** 很多问题前几轮就能从日志看出来。
- **不要把 cron 当队列用。** cron 适合“到点做一次”，高频、密集、需要顺序处理的任务应拆到队列工作流，定时器只负责投递。

## 总结

cron 解决“什么时候做”，heartbeat 解决“每隔多久做一次”。选择依据不是“哪个更简单”，而是任务是否依赖墙钟。定好触发器后，把锁、超时、日志和幂等补上，比纠结表达式更重要。多数 OpenClaw 项目的定时任务故障，不是配置写错，而是用一个语义硬套另一类需求。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/41fd599a4f7ac49f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/4bf6189c0f25781e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/736ea2c777848a70.png)

