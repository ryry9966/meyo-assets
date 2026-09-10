---
title: OpenClaw 的 cron vs heartbeat：两种定时机制怎么选
feedId: 36940
source: 综合讨论
publishedAt: 2026-09-10
---

## 背景

在 OpenClaw 里，凡是"到点干活"的需求，基本落在两条路径上：

- **cron**：调度器驱动，按 cron 表达式、固定间隔或一次性时间点精确触发，任务持久化在本地 jobs store，重启不丢，支持投递到指定频道。
- **heartbeat**：心跳机制，让 agent 按间隔周期性"醒来"，读取 workspace 里的 `HEARTBEAT.md` 巡检清单，自己判断有没有事要做，没事就静默短路。

两者本质上都消耗 agent turn（一次模型调用），但设计意图完全不同，混用是大部分定时任务翻车事故的起点。

## 问题

社区里高频出现的困惑：每天早九点日报该用哪个？巡检服务器、盯未读消息呢？还有典型的踩坑现场——心跳 30 分钟一次把 token 账单拉高一截、cron 没配时区导致日报下午五点才发、任务明明跑了结果却没人收到。根源都是一个：动手前没分清"确定性触发"和"判断性触发"。

## 一句话判断

- **时间点和动作都确定** → cron
- **触发条件模糊，需要 agent 看一眼现场再决定做不做** → heartbeat

## cron 实践步骤

1. 按触发方式选型：`cron`（表达式，如工作日早九）、`every`（固定间隔）、`at`（一次性）。
2. 用 CLI 注册任务（flag 以 `openclaw cron add --help` 为准，下面是示意）：

```bash
openclaw cron add \
  --name "daily-report" \
  --cron "0 9 * * 1-5" \
  --tz "Asia/Shanghai" \
  --prompt "拉取昨日数据，按固定格式输出日报" \
  --deliver --channel telegram
```

3. 显式设置三件事：**时区**、**投递目标**、**超时**。
4. 验证链路：`openclaw cron list` 确认注册状态，`openclaw cron run <id>` 手动触发一次，确认消息真的到达了目标频道。

关键认知：每个 cron job 是独立的 fire-and-forget，prompt 必须自带完整上下文和输出格式，不要依赖"上次对话的记忆"。

## heartbeat 实践步骤

1. 在 workspace 的 `HEARTBEAT.md` 里写一份**短**巡检清单，控制在 10 行以内。
2. 在 openclaw.json 里收紧默认行为：

```json
{
  "agents": {
    "defaults": {
      "heartbeat": {
        "every": "45m",
        "activeHours": { "start": "09:00", "end": "23:00" }
      }
    }
  }
}
```

3. 每次心跳醒来 → 读清单 → 没事就短返回不打扰你，有事才投递到目标频道。

关键认知：无论做不做事，每次醒来都是一次模型调用；且心跳默认跑在 main session 里，塞重活会污染正常对话的上下文。

## 踩坑点

1. **时区**：服务器多为 UTC，cron 不带 tz，"每天 9 点"实际是北京时间 17 点。
2. **心跳烧钱**：默认间隔长开不划算，拉长间隔 + 换更便宜的模型 + 精简清单，三件套做齐。
3. **HEARTBEAT.md 腐化**：清单越写越长，每次心跳成本线性上涨，定期删已完成项。
4. **半夜轰炸**：不设 activeHours，agent 凌晨也会主动发消息。
5. **投递没配好**：任务跑了但结果只躺在 session 日志里，你以为它没干活。
6. **停机补跑**：长时间宕机后，过期的一次性任务通常按 stale 跳过，别假设"开机一定补上"，关键任务部署后要实测一次。

## 可复用建议

- **组合模式**：cron 负责准点输出，heartbeat 负责兜底巡检；可以让 cron job 顺手更新一个状态文件，心跳醒来发现异常状态再跟进，形成闭环。
- **每月清理**：跑一遍 `openclaw cron list`，disable 掉不再需要的僵尸任务。
- **频率纪律**：分钟级的高频纯轮询交给系统级 crontab + 脚本更省，OpenClaw 的定时能力只留给"需要模型理解和判断"的任务。

## 总结

cron 是日程表，heartbeat 是会自省的闹钟。确定时间、确定动作、需要审计 → cron；条件模糊、依赖现场判断、能容忍偶尔漏检 → heartbeat。多数长期运行的部署是两者共存：cron 扛准点业务，heartbeat 低频兜底——重点是别让任何一个越界去干另一个更擅长的活。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/021f1b811ebe61ae.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/c8c25a3b04c50f01.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/53fb2ca910c236ff.png)

