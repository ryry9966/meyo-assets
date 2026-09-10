---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 36859
source: 综合讨论
publishedAt: 2026-09-10
---

## 背景

OpenClaw 里让 agent"自己动起来"的机制有两种：

- **cron**：经典 crontab 语义。写好 cron 表达式和一段 prompt，到点投递给 agent 执行，可以跑在主会话或 isolated session，也能把结果推到指定渠道。
- **heartbeat**：固定间隔的"心跳循环"（默认约 30 分钟，可配置）。每次 tick 会把工作区里的 `HEARTBEAT.md` 注入给 agent，由模型判断"这次要不要干活"，没事就回一个 HEARTBEAT_OK 保持沉默。

一个是你定的闹钟，一个是它的巡更路线。看起来都能"定时"，工程性质完全不同。

## 问题

实际用下来最容易犯的是混用：

- 有人拿 heartbeat 做"每天 9 点发日报"——受 tick 漂移影响时间不准，还每天多烧几十次空闲调用。
- 也有人把"盯某个条件"塞进 cron，每小时全量跑一遍重任务，大部分时间产出为空。

核心判断其实一句话：**任务是时间驱动，还是条件驱动？**

## 做法

**第一步：把任务分两类。**
有明确触发时间（晨报、日报、定时拉取、备份提醒）→ cron；有明确条件、时间不定（文件变化、状态巡检、收件箱检查）→ heartbeat。

**第二步：cron 侧要点。**

```bash
openclaw cron add --name "morning-digest" \
  --cron "0 9 * * *" \
  --session isolated \
  --prompt "读取 xxx，汇总为不超过 10 条的摘要并输出"
```

（具体 flag 以 `openclaw cron --help` 为准。）prompt 必须自包含：写清数据来源、要调用的 MCP 工具、输出格式。isolated session 干净但没有历史上下文，必要背景要写进 prompt。

**第三步：heartbeat 侧要点。**
把 `HEARTBEAT.md` 当检查清单用，控制在 5 条以内，每条写成"条件 → 动作"，并显式声明"没有命中任何条件时直接 HEARTBEAT_OK，不要输出"。间隔按需调大，别用默认值硬扛。

**第四步：组合用法。**
cron 每小时跑一次轻量巡检，把发现写成待办文件；heartbeat 只负责"有待办就处理"。重活归 cron，判断归 heartbeat。

## 踩坑点

1. **heartbeat 的空闲成本**。默认 30 分钟一次，一天约 48 次 LLM 调用，哪怕全是 HEARTBEAT_OK 也计费。先把间隔调到任务能容忍的最大值。
2. **时区**。cron 表达式按宿主机时区解释，容器里 UTC 一跑，"9 点日报"变 17 点是经典事故。上线前先确认 TZ。
3. **session 选择**。isolated 丢历史，main 会和你正在聊的上下文搅在一起，长任务还会阻塞对话。投递型任务优先 isolated + 显式 deliver。
4. **heartbeat 自作主张**。清单写得太模糊（比如"帮我关注下邮件"），模型会扩大解释范围开始乱动作。条件必须可判定，动作必须有边界，破坏性操作要求先确认。
5. **heartbeat 里跑重活**。一次 tick 卡一个 3 分钟的 MCP 调用，整个循环就被拖住了。重活请移交给 cron。

## 可复用建议

- 选型口诀：**看时间点选 cron，看条件选 heartbeat，都要就分层组合**。
- `HEARTBEAT.md` 按产品文档维护，条目越少越好，每周清理一次从未命中的条件。
- 用 `openclaw cron list` 定期审计任务，没用的删掉，任务名要能一眼看懂。
- 心跳只做轻判断，token 用量纳入监控；异常翻倍通常是清单条目开始发散的信号。

## 总结

cron 和 heartbeat 不是竞争关系，是两种不同的触发语义：cron 是确定性时钟，heartbeat 是带判断的环境感知。把"什么时候做"交给 cron，把"要不要做"交给 heartbeat，边界清楚了，agent 才不会在半夜给你发奇怪的消息。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/3ab0ded9b327240b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/efda378b078ba2d2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/bb15a4cd13200362.png)

