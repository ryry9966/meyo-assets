---
title: OpenClaw 定时任务选型：cron 管“到点”，heartbeat 管“间隔”
feedId: 34619
source: 综合讨论
publishedAt: 2026-08-25
---

# OpenClaw 定时任务选型：cron 管“到点”，heartbeat 管“间隔”

## 背景
在 OpenClaw 里跑 Agent、MCP 工具轮询、插件自动化时，定时任务几乎是绕不开的基础能力。openclaw 同时提供 cron 和 heartbeat 两种机制，但不少实践者把它们都理解成“定时触发”，结果出现两种典型误用：用 heartbeat 模拟“每天 02:00 备份”，以及用 cron 秒级表达式去每 30 秒探测一次服务。前者会时间漂移，后者会难维护甚至任务堆积。

## 问题：两种机制的差异没被重视
cron 是日历语义，表达的是“在某个时间点触发”，比如每天 02:00、每周一 09:30。它依赖系统时间和时区，如果上次任务没跑完，下一次仍然会按日历触发。

heartbeat 是间隔语义，表达的是“距离上次触发过去 N 秒/分钟后再次触发”。它不关心现在是几点，只关心节奏是否稳定。因此 heartbeat 更适合健康检查、增量拉取、短周期轮询；而 cron 更适合低峰备份、日报生成、固定窗口任务。

选错机制最常见的后果：

- 用 heartbeat 模拟每日任务：容器重启、进程停顿、任务耗时都会累积偏移，最后触发时间不可控。
- 用 cron 做高频探测：表达式难维护，且 cron 的最小粒度或实现差异可能导致精度不够。
- 长任务重叠：cron 到点就触发，前一次未结束时后一次又启动，产生并发写入或资源争用。

## 做法：先分类，再配置
我在 OpenClaw 里通常先问两个问题：

1. 这个任务关心“具体几点”吗？
2. 这个任务需要“固定节奏”还是“日历时间”？

示例配置如下，简化示意，字段名以你的 OpenClaw 版本为准：

```yaml
tasks:
  nightly_backup:
    type: cron
    cron: "0 2 * * *"
    timezone: "Asia/Shanghai"
    handler: backup_handler
    single_instance: true
    timeout: 3600s

  mcp_health_check:
    type: heartbeat
    interval: 30s
    initial_delay: 10s
    handler: check_mcp_handler
    timeout: 5s
    skip_if_running: true
```

关键点：

- 调度配置里只放触发条件，不写业务逻辑。
- handler 保持轻量，真正耗时操作交给独立进程或队列。
- 为 cron 任务加 `single_instance` 或分布式锁，防止重叠。
- heartbeat 任务设置 `timeout` 小于 `interval`，并明确 `skip_if_running` 策略。

## 踩坑点

### 1. 时区不一致
cron 默认可能是 UTC，国内服务器运行时必须显式设置 `timezone: "Asia/Shanghai"`，否则凌晨任务会跑到白天。

### 2. heartbeat 模拟 cron 的漂移
即使 interval 精确设为 86400s，任务执行耗时、重启、网络抖动都会让下次触发相对上次结束或启动时间滑动。若一定要“每天 02:00”，就用 cron，不要用 heartbeat。

### 3. 长任务重叠
cron 到点触发不等待前一次结束。如果备份耗时超过 cron 周期，必须用锁或 `single_instance`，否则可能同时跑多个备份。

### 4. 高频 heartbeat 堆积
interval 小于 handler 平均耗时是危险配置。比如 10s interval 但任务要 15s，如果实现不跳过，任务会越积越多。应设置 `skip_if_running: true` 或 `timeout` 兜底。

### 5. 多实例重复执行
如果 OpenClaw 部署了多个副本，cron 和 heartbeat 都可能每个副本都触发。需要分布式锁、选主或通过外部调度器统一触发。

### 6. heartbeat 冷启动压力
有些实现启动后立即触发一次，如果 heartbeat 任务较重，会在冷启动时短时间打满资源。建议加 `initial_delay` 或首次只做健康检查。

## 可复用建议

- 决策表：
  - 固定时间点、低峰执行、日报/周报 → cron
  - 每 N 秒/分钟探测、健康检查、增量同步 → heartbeat
  - 需要同时具备“固定窗口”和“高频轮询” → cron 负责全量、heartbeat 负责增量
- 任务幂等：任何定时任务都可能重复执行，写入逻辑必须可重入。
- 监控别只看“有没有触发”，要记录 `last_success_at`，超过阈值告警。
- 调试时先手动执行 handler，再接入调度器，避免调度层问题掩盖业务错误。

## 总结
cron 和 heartbeat 不是谁替代谁的关系。cron 解决“到点执行”，heartbeat 解决“按间隔执行”。把任务语义分清楚，再加好锁、超时、幂等和监控，定时任务在 OpenClaw 里才能稳定运行。对大多数 Agent/MCP 自动化场景，两者组合使用会比强行统一到某一种更省心。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/7c3c57a64a5618c0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/008e7e31da4cbb2a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/90bcace493c0c56d.png)

