---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 36718
source: 综合讨论
publishedAt: 2026-09-09
---

## 背景

OpenClaw 里有两种"到点干活"的机制：cron 和 heartbeat。不少新用户会把两者混着用——要么把所有事都塞进 heartbeat 清单，要么建了一堆 cron job 结果和 heartbeat 重复触发。这篇结合我们节点上跑了几个月的实践，把两者的定位和选择方法讲清楚。

## 本质区别

**cron 是"调度器"语义**：一个 job = 一条自包含的 prompt + 一个精确的时间表达式，到点拉起一次独立的 agent 会话，跑完即止。精确、幂等、可枚举。

**heartbeat 是"心跳循环"语义**：全局只有一个定时器（默认约 30 分钟一跳），每跳给 agent 发一条轻量消息，agent 去读 HEARTBEAT.md 里的清单，自己判断这次有没有值得做的事。它是常驻的环境感知，不是精确闹钟。

一句话：cron 回答"几点做什么"，heartbeat 回答"现在有没有事要做"。

## 怎么选：三步划分法

**第一步，给需求分类：**

- 有确定时刻 + 确定动作的（每天 9 点日报、每周一整理订阅）→ cron
- "帮我盯着，有事说一声"类（盯某个仓库 issue、检查磁盘水位、提醒快到期的待办）→ heartbeat
- 两边都能放的，优先 cron，因为行为可预测、成本可核算

**第二步，配置 cron 时注意三点：** 每条 prompt 写成自包含的（不要依赖上下文）；显式指定投递通道，避免通知乱跑；长任务预估执行时长，避免上次没跑完下次又触发。

**第三步，heartbeat 保持"轻"：** HEARTBEAT.md 只放 3~5 条检查项，并明确写出"无事发生时不输出长内容"；间隔按需调大，30 分钟对多数盯梢场景足够。

## 踩坑点

1. **heartbeat 的 token 成本容易被低估**。即使清单为空，每跳也要跑一次模型调用。见过有人清单写了十几条，一天烧掉的量比正常对话还多。
2. **别用 heartbeat 做精确提醒**。它是"大约每 N 分钟"，不是准点触发。"每天 8:30 提醒我开会"放 heartbeat 必然漂移，必须走 cron。
3. **重复通知**。同一事项同时写在 cron prompt 和 heartbeat 清单里，会收到两遍提醒。定期审计两边，保持互斥。
4. **时区**。cron 表达式按宿主机时区解释，服务器在 UTC 而人在东八区，日报会晚八小时。部署时先确认时区。
5. **清单腐化**。HEARTBEAT.md 用久了越堆越长，单跳耗时和成本线性上涨。每月清一次，把有明确时刻的项迁去 cron。

## 可复用建议

- 决策口诀：**有时刻有动作 → cron；无时刻有状态 → heartbeat**
- HEARTBEAT.md 条目硬上限 5 条，超过就拆
- 每次触发（无论哪种）落一行日志，月末扫一眼，砍掉低价值的 job
- 新需求先进 heartbeat 观察一周，模式和频率稳定后再固化为 cron

## 总结

cron 和 heartbeat 不是替代关系，而是互补：cron 管确定性调度，heartbeat 管概率性巡检。把"精确"交给 cron，把"环境感知"交给 heartbeat，再配合持续的清单和 job 审计，定时任务的成本与行为才能长期可控。如果你有更复杂的划分实践（比如按任务类别路由到不同 agent），欢迎在回帖里分享。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/206dce29bcc09f9c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/e38b751c752b6fa3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/602b09467b671478.png)

