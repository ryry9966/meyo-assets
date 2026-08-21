---
title: OpenClaw 定时任务选型：cron 与 heartbeat 的边界
feedId: 33987
source: 综合讨论
publishedAt: 2026-08-21
---

## 背景

在 OpenClaw 里做自动化任务，定时触发通常有两条路：`cron` 和 `heartbeat`。不少同学把它们都当成“定时器”，实际工程里选错会导致任务重复执行、漏跑，或者周期任务堆积。

简单说：

- `cron` 解决的是“在某个时间点执行”，比如每天 09:00 生成日报。
- `heartbeat` 解决的是“每隔一段时间执行一次”，比如每 5 分钟同步一次邮箱状态。

这两类语义不一样。对 Agent/MCP/插件类自动化来说，区分清楚比写对表达式更重要。

## 问题

选型时最常见的三个坑：

1. **cron 写对了，但时区不对**，任务总在奇怪的时间触发。
2. **heartbeat 任务执行时间超过 interval**，上一个还没跑完，下一个又启动，任务重叠。
3. **没有幂等设计**，失败重试或心跳重叠时产生重复副作用，比如重复发消息、重复写入数据。

下面用一个 OpenClaw 项目里的任务定义示例，说明怎么落地。

## 做法 / 步骤

### 1. 先判断触发语义

如果需求是“每天、每周、每月固定时间”，那就是 `cron`。  
如果需求是“每 N 分钟 / 每小时做一次对账、保活、同步”，那就是 `heartbeat`。

### 2. cron 任务配置

以 OpenClaw 的任务定义为例，建议显式声明时区：

```yaml
tasks:
  - name: daily-digest
    trigger:
      type: cron
      expr: "0 9 * * *"
      timezone: Asia/Shanghai
    action: run_workflow
    params:
      workflow: daily_digest
    guard:
      mutex: true
      retry:
        max: 3
        backoff: exponential
```

`mutex: true` 表示同一任务不允许并发执行。`retry.backoff` 使用指数退避，避免失败后立即重试造成二次压力。

### 3. heartbeat 任务配置

```yaml
tasks:
  - name: mailbox-sync
    trigger:
      type: heartbeat
      interval: 5m
      jitter: 30s
    action: run_workflow
    params:
      workflow: mailbox_sync
    guard:
      mutex: true
      timeout: 4m
```

这里有两个关键点：

- `jitter: 30s` 给触发时间加随机偏移，防止多个实例在同一时刻同时启动，造成惊群。
- `timeout: 4m` 小于 `interval: 5m`，保证任务有足够时间在下一轮触发前结束。

### 4. 执行侧做幂等

无论用哪种触发方式，任务本身最好设计成幂等。比如：

- 用唯一键去重，例如 `task_id + date + target_id`。
- 状态检查后再执行，避免重复写。
- 外部调用设置超时和重试上限。

## 踩坑点

### cron 时区不一致

服务器默认 UTC，而业务预期是本地时间。配置里不写 `timezone`，任务会在 UTC 09:00 触发，而不是北京时间 09:00。排查时先看日志里的触发时间戳。

### heartbeat 任务重叠

如果任务平均耗时 6 分钟，`interval` 却设成 5 分钟，且没有 `mutex`，会出现两个甚至多个实例并发执行。即使有 `mutex`，也可能导致后续触发被跳过或排队。更稳妥的做法是让 `timeout < interval`，并设置 `mutex: true`。

### cron 漏跑与补跑策略

服务重启、停机维护期间，cron 任务可能错过触发窗口。很多调度器默认不会补跑，只在下一次周期触发。如果你的业务要求“错过必须补跑”，需要额外实现 catch-up 逻辑，或者用 heartbeat 做兜底对账。

### heartbeat 重试压垮队列

heartbeat 任务失败后如果每 5 秒重试一次，会让外部服务或 MCP server 承受额外压力。建议使用指数退避，并设置最大重试次数。对于纯检查型任务，可以改为“失败就跳过，等下一个 tick 再试”。

## 可复用建议

- **日历型需求**：每天 09:00、每周一 10:00、每月 1 号 → 用 `cron`。
- **间隔型需求**：每 5 分钟同步、每 10 分钟检查一次 → 用 `heartbeat`。
- **需要精确时间触发** → `cron`，同时显式写 `timezone`。
- **需要保活、对账、同步状态** → `heartbeat`，加上 `jitter` 和 `mutex`。
- **所有定时任务**：默认设置 `mutex: true`，执行逻辑做幂等。
- **监控**：记录每次触发的开始时间、结束时间、触发类型、是否跳过。出现“任务消失”或“重复执行”时，先看触发类型和并发配置。

## 总结

OpenClaw 里 `cron` 和 `heartbeat` 不是简单的“定时器二选一”。  
`cron` 管的是**日历时间点**，`heartbeat` 管的是**运行节律**。

落地时建议遵循：

> 先问“我的任务是某个时间点，还是每隔一段时间”，再决定用哪个触发类型。

然后统一做好三件事：时区、互斥、幂等。这样大部分定时任务都不会踩坑。

---

