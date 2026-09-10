---
title: cron 还是 heartbeat？OpenClaw 定时任务选型实战
feedId: 36965
source: 综合讨论
publishedAt: 2026-09-11
---

## 背景

在 OpenClaw 里让 agent“自己动起来”，官方给了两条路：**cron（定时任务）**和 **heartbeat（心跳）**。很多新用户会混着用，或者干脆把所有周期性需求都塞进 heartbeat，跑一周后发现 token 账单和消息噪音都不对劲。这篇把两者的机制差异和选型思路梳理一下，给个可直接抄的判断框架。

## 两者的机制差异

**cron**：你给一个 cron 表达式（或 `every: 2h` 这类间隔），到点唤醒 agent，执行一段明确的 prompt，跑完把结果投递到指定 channel。行为是确定性的：几点跑、跑什么、发给谁，全是你定的，每次运行通常是独立的上下文。

**heartbeat**：agent 按固定间隔（默认 30 分钟左右，可配）被“戳一下”，醒来后读取一份常驻的说明文件（HEARTBEAT.md），结合上下文自己判断有没有事要做。没事就静默 no-op。它更像“每半小时巡逻一次的值班员”，而不是闹钟。

一句话：cron 是“到点必须干这件具体的事”，heartbeat 是“定期醒来看看要不要做事”。

## 选型做法

1. **先问触发条件是“时间”还是“状态”**：时间驱动（每天 9 点日报、周一晨会周报）→ cron；状态驱动（有新邮件才提醒、指标越界才吭声、待办有变化才整理）→ heartbeat。
2. cron 任务把 prompt 写成**自包含**：不依赖上一次会话的记忆，明确输出格式和投递目标。
3. heartbeat 把判定标准写进 HEARTBEAT.md，且第一条就写“无事发生时保持沉默，不要发消息”。
4. 用 cron 做**兜底**：重要的事不要只靠 heartbeat 的自觉，加一个低频 cron 做强制巡检。

## 踩坑点

- **heartbeat 间隔调太小**：3 分钟一戳，即使 no-op 每次醒来也有基础 token 消耗，一天下来成本可观。建议 30 分钟起步，观察一周再调。
- **HEARTBEAT.md 写得太模糊**：“帮我盯着点事情”会让 agent 自作主张发消息、在群里刷屏。要写成可判定的条件：“仅在 X 发生时通知我，否则不回复”。
- **cron 时区**：宿主机是 UTC，你按北京时间写的 `0 9 * * *` 实际是下午 5 点跑。上线前先确认时区。
- **长任务撞车**：上一轮 cron 还没跑完下一轮又触发，或与交互会话抢同一个 session。给重任务设超时、错开间隔。
- **用 heartbeat 做精确提醒**：心跳间隔 30 分钟，提醒就可能迟到 29 分钟。要准点的，用 cron。

## 可复用建议

- 默认策略：**cron 管确定性的输出型任务，heartbeat 管观察型的守候任务**，组合使用而不是二选一。
- 把 HEARTBEAT.md 当“值班手册”维护：每次 agent 做了不该做的事，就补一条规则进去，迭代式收紧。
- cron 任务一条配置一个职责，prompt 里带当天日期，输出带时间戳，方便排查“到底跑没跑、跑出了什么”。
- 上线前手动立即触发一次，确认投递目标没错，再交给定时器。

## 总结

两者的本质区别是**控制权归属**：cron 把控制权留在你手里，heartbeat 把一部分判断权交给 agent。任务越确定，越该用 cron；越依赖临场判断，才越值得用 heartbeat。先用 cron 建立确定性，再逐步把“需要现场判断”的部分迁到 heartbeat，是最稳的演进路径。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/78723fdb26e8d212.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/7870e744caf26e28.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/637acd15a453ba32.png)

