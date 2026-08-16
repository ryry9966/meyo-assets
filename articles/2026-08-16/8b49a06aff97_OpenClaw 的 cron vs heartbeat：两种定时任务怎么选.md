---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 33362
source: 综合讨论
publishedAt: 2026-08-16
---

## 背景

在 OpenClaw 里做自动化时，定时任务几乎绕不开。常见的触发方式有两种：**cron** 和 **heartbeat**。很多实践者一开始会把它们当成同一个东西：都是“定时跑点逻辑”。但真到了生产环境，容易遇到任务漏跑、重复执行、时区错乱、agent 重启后行为异常等问题。

这篇文章不泛泛讲“定时任务很重要”，只从工程实践角度，把两种机制的边界和选择逻辑说清楚。

## 问题

OpenClaw 中：

- **cron** 基于系统墙钟时间，用 cron 表达式触发，适合“每天 09:00”“每周一 08:30”这类绝对时刻。
- **heartbeat** 基于 agent 生命周期的周期信号，比如每 60 秒一次心跳，适合“每 N 分钟检查一次”“只要 agent 活着就持续做轻量巡检”。

错误的选型通常表现为：

1. 用 cron 模拟高频轮询，导致调度器压力大、任务堆积。
2. 用 heartbeat 承载长耗时任务，阻塞心跳循环，影响 agent 其他能力。
3. 忽略了 cron 的时区和错过补偿，出现任务在错误时间触发或直接漏跑。

下面给出可复用的选择方式和配置思路。

## 做法 / 步骤

### 1. 先判断任务对齐的是“墙钟”还是“运行节奏”

一个简单问题：**这个任务是否需要在一个明确的人类可读时间发生？**

- 需要 → cron
- 只需要固定周期内至少执行一次，不关心具体时刻 → heartbeat

例如：

- 每天 09:00 拉取外部报表、发送摘要 → cron
- 每 5 分钟检查一次内部任务队列、刷新缓存、更新状态 → heartbeat

### 2. cron 配置示例：对齐外部世界

在 OpenClaw 的 agent 配置中，可以给 cron 任务显式设置时区。不要依赖容器默认的 UTC。

```yaml
scheduled_tasks:
  - name: morning_digest
    cron: "0 9 * * *"
    timezone: "Asia/Shanghai"
    action: call_mcp_tool
    tool: news_digest
    args:
      days: 1
```

关键是 `timezone` 显式声明。很多“为什么任务在 17:00 才跑”的问题，其实都是容器里 UTC 与本地时间差了 8 小时。

如果 OpenClaw 支持 `missed_policy`，建议根据业务设置：

- 可以补跑：`catch_up: true`
- 不能补跑：`skip`，但要记录 missed 事件

否则 agent 在目标时刻不可用或重启时，cron 可能静默跳过。

### 3. heartbeat 配置示例：维护运行期节奏

heartbeat 通常不需要完整 cron 表达式，只要一个间隔。

```yaml
heartbeat:
  interval: 300s
  on_tick:
    - check_queue_depth
    - refresh_internal_cache
```

heartbeat 回调一定要**短、幂等、无阻塞**。如果回调里要做耗时操作，应该投递到队列或子任务，不要让 heartbeat 循环等待。

### 4. 混合使用：cron 管外部对齐，heartbeat 管内部健康

比较成熟的形态是：

- 用 cron 触发每天一次的重量级同步，比如通过 MCP 工具拉取外部 API 数据。
- 用 heartbeat 做分钟级轻量检查，比如检测队列积压、agent 状态、缓存过期。

不要让 cron 每 30 秒跑一次，也不要用 heartbeat 执行 10 分钟的模型推理。

## 踩坑点

### 1. cron 时区不是本地时间

容器默认 UTC，即使宿主机是 CST，cron 表达式依然按照进程环境的时间运行。务必在任务配置中显式设置时区，并在日志中打印触发时间，方便核对。

### 2. heartbeat 回调耗时超过 interval

如果 `interval: 60s`，但回调执行 90 秒，可能出现重入或任务堆积。做法是加个轻量级互斥：

```python
if task_running:
    return
task_running = True
try:
    do_work()
finally:
    task_running = False
```

或者使用 `skip_if_running: true`，具体取决于 OpenClaw 的 heartbeat 实现。

### 3. agent 重启后的表现不同

cron 任务在 agent 重启后，如果错过了触发时间，默认可能不补跑。heartbeat 在 agent 启动后会立刻开始新一轮周期，可能导致刚启动就有一波检查。可以在首次 tick 前加 `initial_delay`，或检查上次成功时间。

### 4. 日志刷屏

heartbeat 每次 tick 都记录日志，成功也记，失败也记，很容易淹没重要信息。建议只记录状态变化，例如“连续失败 N 次才告警”“成功不打印，失败打印上下文”。

## 可复用建议

我个人的默认规则：

- **明确时刻 / 每日 / 每周 / 每月** → cron
- **周期内兜底检查 / 常驻巡检 / 相对间隔** → heartbeat
- **任务执行时间可能超过间隔** → 不要用 heartbeat 直接执行，用 cron 或异步队列
- **必须保证不重不漏** → 在设计时明确“错过是否补跑”，并加幂等保护
- **所有定时任务都要带 task_id 和触发时间日志**，否则排障时无法区分是 cron 还是 heartbeat 触发

一个常见的反例是用 cron 每 5 分钟跑一次 MCP 工具检查队列，结果 agent 稍有延迟就会跳过下一次，表现不稳定。换成 heartbeat 后，间隔稍微拉长，反而更稳定，因为 heartbeat 只是生命周期的节奏，不受墙钟对齐的约束。

## 总结

cron 和 heartbeat 并不冲突，它们解决的是不同层次的问题：**cron 负责“世界时间到了就该做”，heartbeat 负责“只要 agent 活着就持续做”**。

选型时先问自己：这个任务是希望在一个具体的时刻发生，还是希望在一个运行周期内至少发生一次？答案会自然指向合适的机制。工程上再做好时区、错过补偿、幂等和日志，定时任务就不会成为半夜把你叫起来的元凶。

---

