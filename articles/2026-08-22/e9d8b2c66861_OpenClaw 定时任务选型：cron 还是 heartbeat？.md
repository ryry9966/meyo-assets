---
title: OpenClaw 定时任务选型：cron 还是 heartbeat？
feedId: 34179
source: 综合讨论
publishedAt: 2026-08-22
---

## 背景

在 OpenClaw 里做自动化，定时触发几乎绕不开两种机制：`cron` 和 `heartbeat`。它们都能让任务自动跑起来，但混用的人不少。有人用 `heartbeat` 每 24 小时执行一次日报，结果时机越来越漂；有人用 `cron` 每 15 分钟巡检队列，却因为执行重叠加剧资源争用。

本文不讨论“哪个更好”，只从调度语义、适用边界和工程实践角度，把选型逻辑理清。

## 问题

核心区别一句话：`cron` 是“到点执行”，`heartbeat` 是“每隔一段时间执行”。

`cron` 依赖绝对时间，受时区和系统时间影响。任务写在 `0 9 * * *`，就是每天 9 点触发。如果进程停机跨过这个时间点，可能直接漏跑。

`heartbeat` 依赖相对间隔，从启动或固定偏移开始计算。配置 `interval: 15m` 后，不关心现在是几点，只关心“上一次到现在是否满 15 分钟”。它没有明确的绝对时间边界，重启后节奏通常会重新开始。

选错典型表现：

- 用 `heartbeat` 做每日汇总，结果执行时间每天后移，慢慢偏离业务窗口。
- 用 `cron` 做高频巡检，执行时间一旦接近间隔，很容易发生任务重叠。
- Agent 场景里 `heartbeat` 间隔太短，频繁唤醒模型，造成 token 和上下文污染。

## 做法

### 1. 先判断是否需要“绝对时间边界”

如果任务描述里包含“每天 9 点”“工作日 18 点前”“每周一”，优先用 `cron`。外部边界、日报、定时发布都属此类。

如果任务只是“每 N 分钟检查一次”“持续保活”“过期检测”，用 `heartbeat` 更自然。它不依赖时钟对齐，适合内部状态收敛。

### 2. 配置示例

OpenClaw 中任务动作建议与调度解耦。动作只做一件事，方便幂等和重放。

```yaml
tasks:
  - name: daily-report
    type: cron
    schedule: "0 9 * * *"
    timezone: Asia/Shanghai
    concurrency: forbid
    misfire: skip
    action: run_agent
    params:
      prompt: "生成今日项目摘要"

  - name: queue-watchdog
    type: heartbeat
    interval: 15m
    start_after: 1m
    jitter: 5s
    action: check_queue
```

> 字段以你的 OpenClaw 版本为准，关键是区分 `cron` 和 `heartbeat` 的调度参数。

### 3. 组合使用

常见组合：`cron` 驱动业务流程，`heartbeat` 做守夜和状态收敛。例如每天 9 点 `cron` 跑日报；每 15 分钟 `heartbeat` 检查任务队列是否卡死。二者不冲突，甚至能互补。

## 踩坑点

- **时区不一致**：容器默认 UTC，业务按北京时间。`cron` 任务不写 `timezone`，可能在凌晨 1 点才触发。务必显式声明。
- **cron 重叠**：任务执行超过触发间隔，多个实例可能同时跑。设置 `concurrency: forbid` 或外部分布式锁。
- **heartbeat 冷启动扎堆**：重启后可能立即触发，和已有 `cron` 任务挤在一起。加 `start_after` 和 `jitter` 错峰。
- **用 heartbeat 代替每日任务**：漂移会累积，且重启会改变节奏。绝对周期边界不要用 `heartbeat` 模拟。
- **cron 错失补偿**：停机跨过触发点，默认可能静默漏跑。需要 `misfire` 策略，例如 `catch_up` 或至少记录日志。
- **Agent 高频 heartbeat**：间隔小于 5-10 分钟，容易打断长期任务、增加无意义模型调用。巡检逻辑尽量轻量，避免每次触发都跑完整 Agent 链。
- **多实例重复触发**：分布式部署下，多个 OpenClaw 实例可能同时响应同一调度。需选主或套锁。

## 可复用建议

- 外部边界用 `cron`，内部巡检用 `heartbeat`。
- 动作与调度解耦，动作保持幂等。
- 给任务加 `run_id` 或幂等键，避免重复执行。
- 观察 p95 执行时长，如果接近 `interval`，立即调大间隔或拆分任务。
- `heartbeat` 维护 `last_success` 时间戳，超过阈值告警，便于发现调度停止或任务卡死。
- 上线前先看时区、`misfire`、`concurrency` 三个字段，很多线上问题都出在这里。

## 总结

`cron` 解决“什么时候做”，`heartbeat` 解决“每隔多久看一眼”。选型关键看任务是否需要绝对时间边界、能否接受漂移、执行是否幂等。实际工程里，多数系统会同时使用两种机制：`cron` 负责外部节拍，`heartbeat` 负责内部保活和异常发现。先把调度语义分清，再配置好时区、锁和错峰，基本能避开大部分定时任务的坑。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/6afceebf93f156af.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/bc717e31565cf7c5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/c135367d9c418cae.png)

