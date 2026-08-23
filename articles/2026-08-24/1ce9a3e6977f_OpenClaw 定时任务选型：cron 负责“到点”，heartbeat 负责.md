---
title: OpenClaw 定时任务选型：cron 负责“到点”，heartbeat 负责“还在不在”
feedId: 34414
source: 综合讨论
publishedAt: 2026-08-24
---

## 背景

在 OpenClaw 里做自动化，定时触发是绕不开的一环。目前常见两类能力：

- **cron**：基于日历时间触发，例如“工作日 09:30 生成摘要”
- **heartbeat**：基于固定间隔触发，例如“每 30 秒检查一次 MCP 连接是否存活”

两者都能做“定时执行”，但语义并不一样。很多实践里被混用的原因很简单：heartbeat 更容易写，就把所有定时任务都塞进 heartbeat。结果是要么重复执行，要么任务间互相阻塞，最后只能把间隔改大，失去实时性。

这篇帖子不讨论泛概念，只围绕 OpenClaw/Agent/MCP/插件场景，给出一个可复现的选型和配置方式。

---

## 问题：什么时候该用哪个

判断标准不是“间隔长短”，而是**触发语义**。

- 如果你关心的是“几点几分做某件事”，用 cron。
- 如果你关心的是“每隔一段时间检查一下状态”，用 heartbeat。

典型 cron 场景：

- 每天 9:00 汇总待办，推送 Agent 简报
- 每小时抓取一次外部数据源
- 工作日傍晚归档当日日志

典型 heartbeat 场景：

- 每 30 秒探活 MCP server
- 每 1 分钟检查任务队列是否有积压
- 每 5 分钟刷新一次本地缓存或 token

一句话：cron 是日历时钟，heartbeat 是节拍器。

---

## 做法 / 步骤

### 1. 先写清楚任务性质

在配置前，给每个定时任务只写一句话：

> 这个任务需要在具体时间发生，还是需要周期性确认状态？

如果答案是“具体时间”，走 cron。答案是“周期确认”，走 heartbeat。这个步骤看起来多余，但能避免后面一半的排障成本。

### 2. cron 配置示例

OpenClaw 的 cron 通常支持表达式、时区和重叠策略。一个可用的配置结构大致如下：

```yaml
schedules:
  - id: daily-digest
    type: cron
    expr: "0 30 9 * * 1-5"
    timezone: Asia/Shanghai
    task: digest
    overlap: skip
```

关键点：

- `timezone` 必须显式设置，否则容易落到 UTC
- `overlap: skip` 表示上一次没跑完时，本次直接跳过
- 任务 ID 保持稳定，方便日志检索和手动补跑

### 3. heartbeat 配置示例

```yaml
heartbeats:
  - id: mcp-watchdog
    interval: 30s
    task: health-check
    overlap: drop
    jitter: 3s
    timeout: 10s
```

关键点：

- `overlap: drop` 避免 tick 堆积
- `jitter` 加随机抖动，防止多实例同时触发
- `timeout` 要小于 `interval`，否则长任务会卡住下一个 tick

### 4. 任务侧保持幂等

无论是 cron 还是 heartbeat，任务本身最好满足：

- 可重复执行，不重复产生副作用
- 有明确结束条件，不无限等待
- 依赖外部服务时设置超时

例如 MCP 工具调用里做健康检查，超时时间应明显小于 heartbeat 间隔，避免 tick 越积越多。

---

## 踩坑点

### 1. 时区问题

cron 默认时区如果未设置，国内用户经常遇到任务晚 8 小时执行。排查时先看触发时间是否正确，再看任务本身。

### 2. 长任务阻塞 heartbeat

这是最常见的问题。heartbeat 每 30 秒触发一次，但任务本身耗时 45 秒。如果没有 `overlap` 策略，后续 tick 会排队，最终导致调度器卡死。

解决方式：

- 任务超时控制
- `overlap: skip` 或 `drop`
- 长任务拆成“入队 + worker 处理”

### 3. cron 与 heartbeat 同时访问同一资源

比如 cron 每天 09:00 做全量缓存刷新，heartbeat 每 1 分钟做缓存探活。两者同时触碰缓存时可能产生竞争。建议共用一把资源锁，或者让 heartbeat 在 cron 执行期间主动让步。

### 4. 多实例重复执行

如果 OpenClaw 部署了多个实例，heartbeat 不设 jitter 时，多个实例会在同一时间点触发同一任务。轻则重复请求，重则放大外部 API 压力。给 heartbeat 加 `jitter`，并对关键任务做单实例锁。

### 5. 改动 cron 后不生效

开发阶段改了 cron 表达式，发现调度器还是按旧时间跑。这多半是调度器缓存或配置未重载。修改后需要确认调度器是否重新加载，必要时手动重启并观察下一次触发日志。

---

## 可复用建议

日常选型可以套这个 checklist：

- 需要“到点就做” → cron
- 需要“每隔一段时间看看” → heartbeat
- heartbeat 间隔建议 30s 到 5min，不要用 1s 做高频轮询
- cron 任务避免超过触发间隔，必要时刻意错开
- 所有定时任务都写幂等
- 日志至少记录：触发时间、开始时间、结束时间、结果状态
- 给 heartbeat 配置 `jitter` 和 `timeout`
- 给 cron 配置 `timezone` 和 `overlap`
- MCP 工具调用类 heartbeat 任务优先做只读探活，写操作放 cron 或队列

---

## 总结

cron 和 heartbeat 不是竞争关系，而是解决两类不同问题：

- cron 解决“日历时间驱动”
- heartbeat 解决“持续状态检查”

实际工程里，常见组合是：cron 负责定时发起批量任务，heartbeat 负责任务执行过程中的健康检查和补偿。不要因为 heartbeat 简单就滥用，也不要因为 cron 精确就把它当轮询器。

选型的关键不是间隔长短，而是触发语义。语义清晰了，配置和排障都会简单很多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/c4339402e4f81fd4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/63cfb939f3eba573.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/2ea2890834193e86.png)

