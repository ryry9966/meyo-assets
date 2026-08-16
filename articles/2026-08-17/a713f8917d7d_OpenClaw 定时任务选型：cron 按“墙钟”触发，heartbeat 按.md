---
title: OpenClaw 定时任务选型：cron 按“墙钟”触发，heartbeat 按“间隔”续命
feedId: 33514
source: 综合讨论
publishedAt: 2026-08-17
---

## 背景

在 OpenClaw 里做自动化时，定时任务基本绕不开两种 trigger：`cron` 和 `heartbeat`。很多人一开始会把它们当成同一种东西，只是配置方式不同。但实际上，它们解决的是两类完全不同的问题：

- **cron**：按“墙钟时间”触发，比如每天 08:30、每周一 09:00。
- **heartbeat**：按“相对间隔”触发，比如每 10 分钟一次、每 6 小时一次。

选错触发方式，轻则任务总在错误时间点执行，重则出现重复执行、漏跑、MCP 工具队列堆积。这篇文章整理一下我在 OpenClaw 里用这两种定时任务的判断方式和配置经验。

## 问题：什么时候该用哪个？

先问自己三个问题：

1. 这个任务是否强依赖某个具体时间点？例如“每天早上 9 点发摘要”“每周五 18 点拉周报”。
2. 如果 OpenClaw 进程重启或休眠，任务错过一次是否可接受？
3. 这个任务是不是一种长期轮询，例如“每 5 分钟看一次队列”“每 30 秒检查一次 MCP 服务状态”。

如果第 1 个问题是“是”，优先 `cron`。  
如果任务的本质是周期轮询、保活、漂移容忍，优先 `heartbeat`。

一个反例：用 `cron` 写 `*/5 * * * *` 去做健康检查，看起来没问题，但它在 OpenClaw 重启后可能不会补跑，且一旦某个时间点卡顿，后续任务容易和上一次重叠。而 `heartbeat` 更适合这种场景。

## 做法/步骤

### 1. cron 配置示例

在 OpenClaw 任务配置中，cron 类型适合明确时间点任务。例如每天 09:00 让 Agent 生成当日待办：

```yaml
tasks:
  - name: morning-digest
    trigger:
      type: cron
      expression: "0 9 * * *"
      timezone: Asia/Shanghai
    action: run_agent
    payload:
      prompt: "基于今天的日历和未读消息，生成 5 条待办摘要"
    timeout: 120s
    retry: 1
```

测试时不要干等到 09:00。可以临时把表达式改成“两分钟后”，例如 `*/2 * * * *`，确认能触发后再恢复。

### 2. heartbeat 配置示例

heartbeat 适合接 MCP 工具或插件做轮询。例如每 10 分钟查一次邮件收件箱，并使用 jitter 避免多个实例同时打到服务：

```yaml
tasks:
  - name: inbox-watch
    trigger:
      type: heartbeat
      interval: 600s
      jitter: 15s
      start_after: 30s
    action: call_tool
    tool: mcp_inbox_fetch
    timeout: 300s
    retry: 0
```

这里 `interval` 控制间隔，`jitter` 是随机抖动，`start_after` 是启动后延迟第一次触发。对于 MCP 工具调用，`timeout` 一定要小于 `interval`，否则容易造成任务重叠或队列堆积。

### 3. 验收方式

- cron：改一个两分钟后触发的表达式，观察日志中的触发时间是否准确。
- heartbeat：启动后观察第一次触发是否延迟了 `start_after`，后续间隔是否稳定，是否存在“上一次还没跑完下一次又来了”的情况。

## 踩坑点

**时区不一致**  
cron 表达式默认可能按 UTC 解释，但业务一般按本地时间。务必显式设置 `timezone`，否则“每天早上 9 点”会变成“下午 17 点”。

**重启后的补偿行为不同**  
cron 在 OpenClaw 停机期间错过的触发通常不会补跑。heartbeat 从进程启动后重新计算，所以在长时间停机后，heartbeat 不会一次性补很多次，但 cron 可能完全跳过。关键业务如果要求补跑，需要在任务内部记录 `last_success_at`，而不要依赖调度器。

**任务重叠**  
cron 在任务执行时间大于表达式间隔时，可能会并发触发同一任务。heartbeat 如果 `timeout > interval`，也会出现类似问题。解决办法：任务做成幂等；如果有共享资源，加锁或用租约。

**heartbeat 被误当成“会话心跳”**  
OpenClaw 里的 heartbeat 是调度器概念，不是 Agent 会话保活。不要用高频 heartbeat 来“维持 Agent 在线”，这会无谓消耗 token，并且可能干扰 Agent 状态。确需保活，建议把间隔拉到分钟级，并且只做轻量探活。

**MCP 工具超时堆积**  
heartbeat 间隔 60 秒，但 MCP 工具 `timeout` 设了 90 秒，这种配置一定会在高峰期堆积。要么调短 `timeout`，要么把 `interval` 调长，要么任务内部做快速失败。

## 可复用建议

- 所有定时任务统一加 `jitter`，错开瞬时并发。
- cron 任务显式写 `timezone`，不要依赖系统默认。
- heartbeat 任务控制 `timeout < interval`，避免堆积。
- 定时任务必须幂等，或者在入口处检查上一次是否完成。
- 为关键定时任务记录 `last_success_at` / `last_failure_at`，用另一个轻量 heartbeat 做监控，而不是只看调度器日志。
- 如果 OpenClaw 配置支持 `retry`，cron 可以少量重试；heartbeat 轮询类任务建议 `retry: 0`，失败等下一轮再处理。

## 总结

cron 解决的是“在某个具体时间点做某事”，heartbeat 解决的是“每隔一段时间做某事”。如果你的任务需要日历语义、错峰执行、和具体时点强绑定，选 cron；如果是后台轮询、状态检查、漂移容忍、接 MCP 工具持续工作，选 heartbeat。

不必纠结术语。只要问一句：**这个任务错过了准确时间点，业务能不能接受？** 能接受，用 heartbeat；不能接受，用 cron。

---

