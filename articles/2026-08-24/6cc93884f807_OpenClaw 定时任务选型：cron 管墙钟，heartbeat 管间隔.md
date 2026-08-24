---
title: OpenClaw 定时任务选型：cron 管墙钟，heartbeat 管间隔
feedId: 34466
source: 综合讨论
publishedAt: 2026-08-24
---

# OpenClaw 定时任务选型：cron 管墙钟，heartbeat 管间隔

## 背景

在 OpenClaw 的任务调度里，cron 和 heartbeat 是两类很容易被混在一起的 trigger。很多 Agent、MCP 工具链或插件场景中，都会出现“看起来都是定时执行”的任务，但两者的调度机制并不一样：cron 依赖墙钟时间，heartbeat 依赖进程内的间隔 tick。

理解这个区别之后，很多看似玄学的“任务偶尔早几分钟/晚几分钟”“重启后任务重复或漏跑”都会变得可解释。

## 问题

选错触发方式，通常会表现为三类现象：

- 固定每天 9 点的日报，偶尔早触发或晚触发；
- 30 秒级的巡检任务被跳过，或者连续触发多次；
- OpenClaw 重启后，定时任务出现补跑、重复或干脆不跑。

本质原因是：没有对齐“任务到底在乎真实时间点，还是在乎固定间隔”。

## 做法/步骤

### 1. 先判断任务的调度语义

如果任务和真实时间有关，例如“每天 9:00 发早报”“每周五 18:00 生成周报”，应该用 cron。

如果任务只关心“隔多久做一次”，例如“每 30 秒消费队列”“每 5 分钟做健康检查”，应该用 heartbeat。

### 2. 在 OpenClaw 任务定义中显式声明 trigger

示例配置如下，字段名以你当前 OpenClaw 版本的 schema 为准：

```yaml
tasks:
  - name: daily-digest
    trigger:
      type: cron
      expr: "0 9 * * *"
      timezone: "Asia/Shanghai"
    action: digest.publish
    options:
      timeout: 10m
      concurrency: forbid

  - name: queue-watch
    trigger:
      type: heartbeat
      interval: 30s
      jitter: 5s
      max_interval: 60s
    action: queue.consume
    options:
      timeout: 25s
      concurrency: forbid
```

### 3. 做短周期验证

不要一上来就挂到正式任务上。可以先把 cron 表达式改成 1 分钟内的测试时间，把 heartbeat 的 interval 改成 10 秒，观察日志里的触发时间、duration、last_run_at 是否稳定。

### 4. 测试重启恢复

停掉 OpenClaw 运行器，等一个调度周期后再启动，观察任务是否补跑、是否重复、是否从当前时间重新计时。这一步能暴露很多调度器补偿策略的差异。

### 5. 统一 timeout、retry、concurrency

不要每个任务各写一套。把超时、重试、并发策略固定下来，后续排障会简单很多。

## 踩坑点

### 1. 用 cron 表达“每 30 秒一次”不可靠

cron 的分钟粒度、调度器补偿机制和 tick 语义，都决定了它不适合高频短间隔任务。遇到这种需求，直接换 heartbeat。

### 2. heartbeat 不保证墙钟对齐

进程重启、Agent 主循环阻塞、长时间 action，都会让下一次 tick 推迟。不要用 heartbeat 做“每天 9 点”这种任务。

### 3. 重叠执行

如果 action 执行时间大于 heartbeat interval，默认情况下可能拉起多个实例。务必设置 `concurrency: forbid`，或者使用分布式锁/幂等键。

### 4. cron 的时区与补跑

cron 可能按 UTC 解析，导致本地时间偏差。部分调度器在重启后还会补跑错过的任务，容易把原本已经失败或重试过的任务再次拉起。

### 5. heartbeat 的“无心跳”问题

OpenClaw 进程卡死时，heartbeat 不会触发，也没有 cron 那样的补跑机制。对关键 heartbeat 任务，应加 `max_interval` 或外部 watchdog。

## 可复用建议

- 建立两个配置模板：`cron-task` 和 `heartbeat-task`，统一 timeout、retry、concurrency、日志字段。
- 决策表：固定墙钟 → cron；固定间隔/高频 → heartbeat；固定墙钟 + 持续消费 → cron 作为入口，内部再用 heartbeat 循环。
- 给每个定时任务记录 `last_run_at`、`duration_ms`、`overlap_count`。出现漂移时，先看调度语义，而不是急着加更多定时器。

## 总结

cron 是“到点叫我”，heartbeat 是“每隔一段叫我”。OpenClaw 里没有哪个更好，只有是否匹配任务的真实时间语义和间隔语义。

选型时先问自己一句：这个任务的触发条件，是由墙钟定义的，还是由运行节奏定义的。把这个问题回答清楚，大部分漂移、重复和漏跑都会少很多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/b989a43f780cf15c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/3da8421e72d4f9ac.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/2ac7aa471c927b5b.png)

