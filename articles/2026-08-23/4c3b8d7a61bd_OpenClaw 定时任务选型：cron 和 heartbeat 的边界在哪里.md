---
title: OpenClaw 定时任务选型：cron 和 heartbeat 的边界在哪里
feedId: 34393
source: 综合讨论
publishedAt: 2026-08-23
---

## 背景

OpenClaw 里很多自动化需求都挂在“定时”上：早上发一次摘要、每 30 秒巡检一次 MCP server、每隔一段时间清理上下文、定时调用某个插件同步数据。社区里最常见到的配置是 `cron` 和 `heartbeat`，但不少人把两者当同类调度器互换使用，结果不是重复触发，就是错过窗口。

这不是“哪个更好”的问题，而是两类触发模型。

## 问题

`cron` 是**对齐时钟**的触发：任务在某个日历语义点执行，比如每天 8:00、每周一 09:30。它关心的是“现在是不是该到点了”。

`heartbeat` 是**固定节奏**的触发：任务以相对时间间隔反复 tick，比如每 60 秒醒来一次。它关心的是“距离上次有没有过够一个 interval”。

如果搞混，会出现两类典型症状：

- 用 heartbeat 模拟“每天早上 8 点跑”，结果 tick 漂移、进程阻塞后越跑越偏；
- 用 cron 做“每 30 秒探活一次”，结果频率太粗，或者产生大量空跑。

## 做法 / 步骤

先判断任务是否依赖绝对时间。

**1. 有日历语义，选 cron**

例如：“每早 9 点生成摘要”“每周一上午同步订阅”“每月 1 号清理归档”。

这类任务适合写成一个普通的 cron job。以 OpenClaw 的任务配置为例，简化片段如下：

```yaml
tasks:
  - name: morning-digest
    type: cron
    schedule: "0 9 * * *"
    timezone: Asia/Shanghai
    action: agent.run
    params:
      skill: digest
    timeout: 120s
    retry: 2
```

需要注意：`timezone` 一定要显式声明，否则容器默认 UTC，容易导致中国区任务下午才跑。

**2. 只有相对节奏，选 heartbeat**

例如：“每 60 秒检查一次任务队列”“每 10 分钟 ping 一次 MCP server”“每 5 分钟刷新一次插件状态”。

这类任务适合 heartbeat：

```yaml
tasks:
  - name: mcp-healthcheck
    type: heartbeat
    interval: 60s
    max_concurrency: 1
    timeout: 50s
    action: mcp.ping
    params:
      server: main
```

核心原则是：`timeout` 应小于 `interval`，避免任务还没结束，下个 tick 又进来。

**3. 给所有定时任务加执行边界**

不管选哪一种，都建议给任务配置：

- 幂等 key：防止补跑或重试时重复处理；
- 明确超时：尤其 agent 调用和 MCP 调用；
- 失败通知：不要静默失败；
- 日志字段：`last_run_at`、`last_success_at`，否则排障很难。

## 踩坑点

### cron 不一定补跑

OpenClaw 的 cron 在进程错过窗口后，默认不会补跑。比如容器重启、系统休眠期间正好跨越了任务时间，恢复后不会集中追补。

这在多数场景是对的，避免恢复时一股脑触发。但如果业务允许补跑，需要打开补跑参数，同时必须保证任务幂等。否则补跑一次就得手工处理一批重复数据。

### heartbeat 最容易重叠执行

尤其是 agent 任务执行时间不稳定。60 秒 interval，但上轮跑了 90 秒，下轮已在队列里。如果没有 `max_concurrency: 1` 或分布式锁，可能两个 tick 同时操作同一资源。

单实例场景设置 `max_concurrency: 1` 即可；多实例部署必须引入 leader election 或外部锁。

### 不要用 heartbeat 模拟 cron

“每分钟检查一下现在是不是 9 点”不可靠。tick 会漂，进程阻塞会推迟，边界时间会错过。需要日历语义时，直接用 cron。

### 分布式部署时两者都要防重

多实例跑 OpenClaw 时，cron 和 heartbeat 都可能被多个实例同时触发。heartbeat 尤其明显。除了配置并发限制，建议关键任务走外部调度或带锁队列。

## 可复用建议

- **固定时间点 / 日历语义 / 对外通知类**：选 `cron`；
- **相对周期 / 持续巡检 / 状态同步类**：选 `heartbeat`；
- **长时间任务**：不要直接放在 heartbeat 里裸跑，要么走队列，要么用 cron 作为入口、heartbeat 做监督；
- **所有定时任务都保持幂等**：不要依赖“它应该只跑一次”；
- **把 schedule 与 action 分开配置**：先手动触发 action 验证，再挂定时，减少排障面；
- **关键任务不要只依赖进程内调度**：外部监控或兜底任务会更可靠。

## 总结

cron 解决的是“到点了没有”，heartbeat 解决的是“节奏还对吗”。选择哪一类，取决于任务是否依赖绝对时间。工程上更稳妥的做法是：触发层选对模型，执行层做好幂等、超时和锁，别让定时任务变成线上排障的盲区。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/b21a5ee7cf0a1a6f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/fda86df795f223a0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/d4bd943d06e202a0.png)

