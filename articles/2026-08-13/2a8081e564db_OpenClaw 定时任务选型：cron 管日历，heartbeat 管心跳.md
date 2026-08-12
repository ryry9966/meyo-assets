---
title: OpenClaw 定时任务选型：cron 管日历，heartbeat 管心跳
feedId: 32842
source: 综合讨论
publishedAt: 2026-08-13
---

## 背景

在 OpenClaw 里接自动化、MCP 工具或插件时，定时任务经常不是“定个时间跑一跑”这么简单。调度方式会直接影响任务堆积、时区偏移、重启恢复，甚至 MCP 连接被误判为失活。

社区里最常见的是两种配置：`cron` 表达式和 `heartbeat` 固定间隔。它们看起来都能定时触发，但语义完全不同。选错之后，要么任务在错误的时间点执行，要么同一任务被并发跑多次，排查起来比业务逻辑还费劲。

## 问题

- 该按自然时间跑的任务，如果用 heartbeat，会因为固定间隔和启动时间发生漂移。比如“每天早上 8:30 发日报”，heartbeat 每 6 小时跑一次，第一次 8:30，第二次可能 14:30，第三次 20:30，第二天变成 2:30。
- 需要持续巡检、续约、消费队列的任务，如果用 cron，会在两次 cron 之间留下盲区。比如“每 1 分钟检查一次队列”，cron 只能做到分钟级触发，且任务如果超过 1 分钟，下一轮可能还没结束上一轮就来了。

简单说：**cron 解决“什么时间做”，heartbeat 解决“是否持续在做”**。

## 做法/步骤

### 1. 先判断任务类型

写下任务需求时，先问一句：这个任务是日历对齐，还是存活/巡检？

- “每天早上 8:30 生成 digest”“每小时抓一次外部 API”——日历任务，用 cron。
- “每 30 秒检查 MCP 队列”“每 10 秒续约临时凭证”“每分钟探活 downstream 服务”——存活/巡检任务，用 heartbeat。

### 2. cron 配置示例

OpenClaw 的任务配置可能类似这样，具体字段以你所用版本的 schema 为准：

```yaml
tasks:
  - name: daily-digest
    type: cron
    expression: "30 8 * * *"
    timezone: Asia/Shanghai
    overlap: skip
    run: mcp.digest.send
```

需要注意：

- `expression` 一般使用 5 段标准 cron，从分钟开始。不要直接把 Quartz 的 6 段带秒表达式套进来，部分解析器不兼容。
- `timezone` 必须显式声明。容器默认常是 UTC，不写就很容易晚 8 小时。
- `overlap: skip` 或等价并发策略要加上。如果任务执行时间可能超过下一轮触发时间，建议用 `skip` 而不是默认并发。

### 3. heartbeat 配置示例

```yaml
tasks:
  - name: queue-watch
    type: heartbeat
    interval: 30s
    jitter: 5s
    overlap: skip
    run: mcp.queue.consume
```

说明：

- `interval` 不要小于任务平均执行时间，否则很容易堆积。
- `jitter` 是多实例部署时防止所有节点同时触发的关键，建议给一个随机抖动。
- 失败退避可以设 `max_backoff` 或等价字段，避免异常时高频重试把下游打坏。

## 踩坑点

1. **cron 时区偏移**  
   容器内 `date` 显示正确，但 cron 仍按 UTC 跑。因为系统时区和应用解析 expression 的时区不是一回事。配置里不写 `timezone`，就会出现晚 8 小时或早 8 小时。

2. **heartbeat 任务并发叠加**  
   heartbeat 间隔 30 秒，任务平均执行 40 秒，如果不限制并发，第二轮会在第一轮未结束时启动，导致队列消费重复或锁竞争。`overlap: skip` 或分布式锁二选一。

3. **cron 重启错过不补偿**  
   如果服务在触发点重启，或触发时正好发布，cron 任务不会补跑。关键任务需要做对账或显式 miss-policy，不要假设它一定执行。

4. **heartbeat 空转浪费资源**  
   每 30 秒查一次数据库或外部 API，连接池可能被占满。heartbeat 应只做轻量检查或状态标记，重查询交给 cron 或事件驱动。

5. **表达式兼容性**  
   拿 Spring/Quartz 的 6 段表达式直接放到 OpenClaw 里，可能不生效或直接报错。先确认解析器支持几段，再写配置。

## 可复用建议

- **用 cron 的场景**：日报、定时抓取、与外部系统按自然时间对齐的重任务。
- **用 heartbeat 的场景**：租约续期、健康检查、队列消费、状态同步、连接保活。
- **混合使用**：heartbeat 只做“检测和标记”，真正重任务交给 cron 或任务队列。这样 heartbeat 保持轻量，cron 保持日历语义。
- **配置集中管理**：把 `timezone` 和 `expression` 放在同一段配置里，避免跨文件漂移。
- **默认加 overlap 策略**：无论 cron 还是 heartbeat，都显式声明并发策略，不要依赖默认值。

## 总结

在 OpenClaw 里，cron 和 heartbeat 不是同一种东西的两种写法，而是两种调度语义。日历对齐、精确到某时某分执行的任务选 cron；需要持续探测、续约、消费、保活的任务选 heartbeat。选定之后，把时区、并发策略、失败退避这些工程细节配好，大概率能避开社区里最常见的定时任务坑。

---

