---
title: OpenClaw 定时任务选型：cron 负责“几点做”，heartbeat 负责“多久看一次”
feedId: 34118
source: 综合讨论
publishedAt: 2026-08-22
---

# OpenClaw 定时任务选型：cron 负责“几点做”，heartbeat 负责“多久看一次”

在 OpenClaw 里做自动化，很容易同时面对两种定时触发：`cron` 和 `heartbeat`。它们都能“定时跑”，但设计语义完全不同。选错不会立刻报错，通常会在某次重启、时区切换或任务变慢后才暴露。

## 背景

OpenClaw 的 task/automation 配置里，触发器通常支持两类：

- `cron`：日历语义，按表达式在指定时间点触发。
- `heartbeat`：间隔语义，按固定间隔重复触发。

很多场景里两者都能凑合，比如“每 24 小时跑一次”可以用 `heartbeat: 86400s`，也可以用 `cron: 0 9 * * *`。但工程上这不是等价选择。

## 问题：两种语义为什么不等价

`cron` 回答的是“什么时候做”，它绑定的是钟表时间。它天然适合日报、定时发布、对账等带有业务时间概念的任务。

`heartbeat` 回答的是“每隔多久看一次”，它绑定的是进程启动后的相对时间。它更适合健康检查、队列消费、缓存刷新、状态巡检这类“不需要关心现在几点，只需要周期性执行”的任务。

常见的误用是把 `heartbeat` 当成 `cron` 用。比如想每天 09:00 跑一次，却配置 `heartbeat: 24h`。如果 OpenClaw 实例在 14:30 启动，第一次触发就是次日 14:30，之后持续偏移。容器重启后还会重新计算起点。

反过来，把 `cron` 当成高频轮询也有问题。比如每 10 秒轮询一次任务队列，用 `cron` 表达式很难表达秒级频率，且调度器容易产生额外延迟。

## 做法

### 1. 先判断任务语义

写配置前先问一句：如果实例重启，触发时间是否应该继续锚定在某个具体时间？

- 是：用 `cron`。
- 否：用 `heartbeat`。

### 2. cron 配置示例

```yaml
tasks:
  - name: daily-report
    trigger:
      type: cron
      expr: "0 9 * * *"
      timezone: "Asia/Shanghai"
    action:
      type: agent
      prompt: "生成昨日运行摘要"
    options:
      catch_up: false
      timeout: 120s
```

要点：

- `timezone` 明确设为本地时区，避免容器默认 UTC。
- `catch_up: false`：如果实例在触发点不可用，补跑与否要显式决定。日报类通常关闭补跑，避免一次性涌入历史任务。

### 3. heartbeat 配置示例

```yaml
tasks:
  - name: queue-watch
    trigger:
      type: heartbeat
      interval: 60s
      jitter: 5s
    action:
      type: plugin
      name: check_queue
    options:
      timeout: 30s
      max_concurrent: 1
```

要点：

- `jitter` 用于把触发时间打散，避免多实例或同机多任务同时启动造成惊群。
- `timeout` 必须小于 `interval`，否则任务一旦变慢就会开始积压。
- `max_concurrent: 1` 防止上一次未结束，下一次又进入。

## 踩坑点

1. **时区**：`cron` 不配时区，默认 UTC。国内用户看到“09:00 的任务为什么 17:00 才跑”，先查时区。
2. **重叠执行**：`heartbeat` 间隔小于任务执行时间时，可能产生堆叠。没有 `max_concurrent` 或锁时，任务可能把下游打满。
3. **重启偏移**：`heartbeat` 的起点通常与启动时间相关。不要用它实现“每天固定时间”。
4. **无 jitter 惊群**：多副本部署时，同一间隔的 heartbeat 往往同时触发，造成数据库或 API 瞬时尖峰。
5. **失败重试与幂等**：定时任务没有幂等设计时，重复触发会产生重复数据。尤其 `catch_up: true` 或重试时更明显。

## 可复用建议

- 日报、定时发布、定时对账：`cron` + 固定时区 + 关闭补跑。
- 健康检查、队列消费、状态刷新：`heartbeat` + `jitter` + `timeout` + 单并发。
- 需要“每几秒”的高频任务：优先 heartbeat，不要用 cron 硬凑。
- 给所有定时任务加两个监控指标：上一次触发时间、上一次执行耗时。若 heartbeat 触发间隔明显漂移，通常说明任务执行超过间隔或调度器被阻塞。
- 把任务逻辑写成幂等：以日期、批次号或唯一键去重，重试不会产生脏数据。

## 总结

在 OpenClaw 里，`cron` 和 `heartbeat` 不是同一个东西的两种写法，而是两种触发模型。前者锚定日历时间，适合业务周期；后者锚定运行间隔，适合守护与巡检。选型时优先看任务是否关心“现在几点”。如果关心，用 cron；如果只关心“每隔多久看一下”，用 heartbeat。然后补上时区、超时、jitter 和幂等，比单纯调通触发配置更重要。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/3ea63c3dbb2b5ead.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/dbbae26537efaa6f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/9339ee4a35a91bd4.png)

