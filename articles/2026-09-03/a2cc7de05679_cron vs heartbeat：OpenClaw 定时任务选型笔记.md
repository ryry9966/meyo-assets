---
title: cron vs heartbeat：OpenClaw 定时任务选型笔记
feedId: 35915
source: 综合讨论
publishedAt: 2026-09-03
---

## 背景

在 OpenClaw 里让 agent"定期干活"，有两条路径：**cron**（按时间表达式精确触发，运行在隔离上下文，可显式配置投递目标）和 **heartbeat**（agent 主循环按固定间隔醒来，扫一遍工作区的 `HEARTBEAT.md`，在当前会话里处理）。很多人一开始混着用，结果要么定点任务漏跑，要么空转 tick 把 token 用量翻倍。这篇帖基于社区几个真实案例，把选型边界讲清楚。

## 问题

典型纠结场景：

- 每天 9:00 汇总 issue 并推送到群——用哪个？错过 9 点怎么办？
- "agent 有空时看看有没有人 @ 我"——这种模糊任务放 cron 里，时间表达式写什么？
- 同一个任务两边都挂了，一天被推送两次。

本质是三个维度没对齐：**时间精度、上下文依赖、失败语义**。

## 先看差异

| 维度 | cron | heartbeat |
|---|---|---|
| 触发 | 时间表达式，精确到分钟 | 固定间隔 tick |
| 上下文 | 隔离、自包含 | 共享当前会话/工作区 |
| 投递 | 显式声明目标 | agent 自行判断 |
| 错过触发 | 有跳过/补跑策略 | 下个 tick 自然继续 |
| 适合 | 定点日报、定时推送 | 环境感知、待办清扫 |

## 做法与步骤

**第一步：给任务分类**，回答两个问题——触发时刻必须精确吗（误差容忍 < 5 分钟）？执行时需要当前会话上下文吗？

**第二步：精确且不依赖会话的任务走 cron**（示例，参数以你的版本为准）：

```bash
openclaw cron add \
  --name daily-issue-digest \
  --schedule "0 9 * * 1-5" \
  --prompt "拉取仓库过去 24h 新 issue，按优先级归纳，推送到 #dev 频道" \
  --deliver channel
```

要点：prompt 必须自包含，投递目标显式声明。

**第三步：模糊时机、依赖上下文的任务进 heartbeat**，写进工作区的 `HEARTBEAT.md`：

```markdown
## 待巡检
- [ ] 有未回复的 mention 就整理成列表；没有则不要输出
```

**第四步：验证**。cron 手动触发跑一次看投递链路；heartbeat 把间隔临时调到 1m，观察 2–3 个 tick 再调回生产值。

## 踩坑点

- **heartbeat 空转烧 token**：每个 tick 都有开销。任务少就拉长间隔，并在任务描述里明确"无事不要输出"。
- **cron 是隔离上下文**：别指望它记得昨晚的对话，所需信息全部写进 prompt。
- **时区**：容器里默认 UTC，cron 表达式按本地时间理解会整体偏移，先 `date` 确认再写表达式。
- **错过触发的语义不同**：cron 有补跑/跳过策略，heartbeat 只是下个 tick 继续。硬截止任务不要放 heartbeat。
- **双重触发**：同一任务同时在 cron 和 heartbeat 里，明确唯一归属，另一边最多留一行指针。

## 可复用建议

- 一句话决策：**精确时间或需要对外投递 → cron；模糊时机、需要上下文判断 → heartbeat；长任务两者都不合适，走 workflow/队列。**
- `HEARTBEAT.md` 保持精简，常驻不超过 5 条，每周清理已完成项。
- cron 任务统一命名规范（频率-动作-目标），方便审计和批量下线。
- 上线前把间隔调小做压力验证，确认后再恢复。

## 总结

cron 和 heartbeat 不是替代关系，而是覆盖不同的容差区间：cron 管"准点"，heartbeat 管"在场"。把时间精度和上下文依赖作为分类依据，绝大多数选型纠结会在两分钟内消失。欢迎在评论区贴出你们的具体任务场景，一起对齐判断标准。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/ef19086556ba47d3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/15cc85d90abf8bf4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/7c7060c5ecd23583.png)

