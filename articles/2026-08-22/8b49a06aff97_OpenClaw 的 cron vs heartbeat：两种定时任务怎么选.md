---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 34126
source: 综合讨论
publishedAt: 2026-08-22
---

# OpenClaw 的 cron vs heartbeat：两种定时任务怎么选

在 OpenClaw 里挂定时任务时，经常会看到两个入口：cron 和 heartbeat。它们都能让 Agent 在后台周期性执行逻辑，但面向的问题不同。选错轻则任务执行时机不对，重则任务重叠、重复上报或把外部 API 打爆。

## 背景：两种触发器解决的是不同的问题

cron 是日历型触发器，回答“在什么时间点执行”。比如每天 9:00 汇总一次数据，每周一生成周报。它依赖系统或运行时的 cron 表达式，任务触发时间可预期、可对齐。

heartbeat 是间隔型触发器，回答“每隔多久跑一次”。比如每 30 秒检查一次任务队列，每 5 分钟同步一次 Agent 状态。它不关心当前是几点，只关心距离上次触发过了多久。

在 OpenClaw 的 agent 或 MCP 插件里，这两种触发器通常可以共存，但不要把它们当成可互换配置。

## 问题：选错触发器会怎样？

典型问题有三类：

1. 用 heartbeat 做日历任务：比如“每天 9 点发日报”，如果写成 heartbeat `86400s`，一旦 Agent 重启或延迟启动，执行时间会整体漂移。而且 heartbeat 的起点是进程启动时间，不是自然日。
2. 用 cron 做高频状态检查：比如每 30 秒检查一次 queue，cron 表达式最小粒度通常到分钟，且反复解析表达式、对齐时钟会带来不必要的复杂度。
3. 两个触发器同时驱动同一个任务：配置分散后容易重复执行，尤其在多实例部署时更明显。

## 做法：先判断任务性质，再选触发器

比较稳妥的选型步骤如下：

1. 问自己：这个任务和“自然时间/日历”有关吗？
   - 有关 → 用 cron。
   - 无关，只和“距离上次执行多久”有关 → 用 heartbeat。

2. 问自己：任务是否幂等？
   - 不幂等的任务，避免高频 heartbeat 或短间隔 cron，防止并发执行。
   - 幂等任务可以放宽，但仍建议做防重入。

3. 问自己：执行时长是否可能超过触发间隔？
   - 可能超过 → heartbeat 必须加锁或 skip if running；cron 则要评估错过触发后的补跑策略。

以 OpenClaw 为例，常见配置大致是这样的（配置片段省略具体字段，以你当前版本为准）：

```yaml
# 日历型：每个工作日 9 点生成前一日摘要
triggers:
  - name: daily_digest
    type: cron
    schedule: "0 9 * * 1-5"
    action: run_agent_digest

# 状态型：每 10 分钟检查一次未处理任务数
  - name: queue_check
    type: heartbeat
    interval: 600s
    action: check_queue_depth
```

配置完成后，先在本地把触发频率调低跑一轮，确认 action 能正常返回，再恢复正式间隔。

## 踩坑点

1. **时区错位**：容器里 cron 默认可能是 UTC。如果你在中文环境，写 `0 9 * * *` 实际会在北京时间 17 点触发。务必在 runtime 配置里设置 `TZ=Asia/Shanghai`，或使用带时区的 cron 实现。

2. **heartbeat 启动即触发**：很多 heartbeat 实现会在 agent 启动后立即触发一次。如果这时候依赖的资源还没初始化（比如数据库连接、MCP server 未注册），会直接报错。需要加“首次延迟”或就绪检查。

3. **任务重叠**：heartbeat 间隔 30 秒，但任务执行了 50 秒，下一次触发时上一次还没结束。如果 action 没有锁，就容易出现并发写冲突。建议用 `if_running: skip` 或分布式锁。

4. **cron 错过触发**：Agent 在计划时间点正好重启或宕机，cron 可能直接跳过。对于重要任务，需要补跑策略，比如记录 last_success_at，启动时检查是否漏跑。

5. **多实例重复触发**：多个 OpenClaw 实例同时运行同一条 heartbeat，每个实例都会触发。需要用 leader election 或把任务拆分到固定实例。

## 可复用建议

- 给每个定时任务加 `name` 和日志标签，不要用匿名 action。出问题时你能快速定位是哪个 trigger 驱动。
- heartbeat 任务尽量幂等，且间隔不要小于实际执行时长。
- 对日历型任务，cron 表达式统一用字符串配置，避免用“每 86400 秒”这种隐式日历。
- 给 heartbeat 加 1-5 秒的 jitter，尤其在有多个实例或同时启动的场景，能避免瞬时请求尖峰。
- 维护一个简单的执行记录：`last_start_at`、`last_success_at`、`last_error`。这对排查和补跑非常有用。

## 总结

cron 和 heartbeat 不是谁更好的问题，而是任务语义不同。简单说：

| 维度 | cron | heartbeat |
|---|---|---|
| 关注点 | 自然时间点 | 固定间隔 |
| 典型场景 | 日报、周报、定时同步 | 队列检查、心跳保活、状态轮询 |
| 时间精度 | 通常分钟级 | 秒级或分钟级 |
| 主要风险 | 时区、错过触发 | 任务重叠、多实例重复 |

在 OpenClaw 里做自动化，别把 cron 当“长间隔 heartbeat”用，也别把 heartbeat 当“短间隔 cron”用。先想清楚任务和日历有没有关系，再决定配置哪种触发器，能少踩很多坑。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/4143f5b73d00092b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/3146b6a6fc604f25.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/dd77cf7533746e92.png)

