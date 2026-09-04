---
title: cron vs heartbeat：OpenClaw 定时任务的分工、配置与踩坑
feedId: 36026
source: 综合讨论
publishedAt: 2026-09-04
---

## 背景

OpenClaw 里有两套"定时"机制，群里反复被问到怎么选：

- **heartbeat（心跳）**：网关按固定间隔（默认 30 分钟，`agents.defaults.heartbeat.every` 可调）唤醒 agent，让它读工作区里的 `HEARTBEAT.md`，由 agent 自己判断有没有事做，没事就静默跳过。
- **cron（定时任务）**：经典调度器，支持 cron 表达式、固定间隔和一次性任务，每个任务带明确 payload，跑在独立会话里，可把结果投递到指定频道。

两者看起来都能"定时干活"，但设计意图完全不同。

## 问题

我自己踩过的两个典型坑：

1. 把一堆巡检项全塞进 `HEARTBEAT.md`，每次心跳全量跑一遍，token 消耗直接翻倍，主会话还多了一堆无关上下文；
2. 用心跳做"每天 9 点发日报"，结果实际触发时间飘到 9:07、9:23——心跳是"间隔唤醒 + agent 忙时跳过"，本来就不是精确闹钟。

## 怎么选、怎么做

一句话：**到点必须发生的用 cron，"没事别吵我"的巡检用 heartbeat。**

**第 1 步：需求分三类。** 精确时刻或一次性提醒 → cron；低频条件巡检（有异常才说话）→ heartbeat；外部事件触发 → 走 webhook 类通道，别在这两者里硬凑。

**第 2 步：配置 heartbeat。** 工作区建 `HEARTBEAT.md`，控制在几行以内，并写明无事时的退出条件：

```markdown
- 检查 ~/logs/app.log 今天是否有 ERROR
- 检查 git 工作区是否有未提交的关键改动
- 全部正常：不做任何事，直接结束
```

间隔从默认 30m 起步，确实不够再调，别一上来就压到 5m。

**第 3 步：配置 cron。**

```bash
# 工作日 9:00 日报：独立会话运行，结果投递到 Telegram
openclaw cron add --name "daily-report" \
  --cron "0 9 * * 1-5" \
  --message "汇总昨天的会话与待办，生成日报" \
  --deliver --channel telegram --to <chat_id>

# 一次性提醒
openclaw cron add --name "deploy-reminder" \
  --at "2025-06-01 14:00" \
  --message "提醒我确认发布窗口"
```

具体 flag 以当前版本的 `openclaw cron add --help` 为准。

**第 4 步：验证。** cron 用 `openclaw cron list` 确认注册结果，手动 `cron run` 跑一次验证投递链路；heartbeat 看网关日志，确认无事时确实静默、有事时才发言。

## 踩坑点

1. **拿心跳当闹钟**：agent 忙或会话活跃时心跳会被跳过/延后，天然有抖动，定时发布、准点提醒一律走 cron。
2. **`HEARTBEAT.md` 写成长清单**：每次唤醒都是一次完整的 agent 运行，条目越多越烧钱，只放轻量检查和明确的"无事退出"。
3. **cron 时区**：服务器多是 UTC，按北京时间定任务要先确认网关时区或手动换算，`--at` 一次性任务同理。
4. **cron 结果无脑投递到群**：日报、报警才 deliver；后台维护类任务留在独立会话，别刷屏。
5. **重复执行**：网关重启后错过的任务可能补跑，message 里带上日期等幂等要素，避免同一条报警发两遍。

## 可复用建议

- 任务命名带用途和频率，如 `daily-report-0900`，定期 `openclaw cron list` 清理僵尸任务；
- 巡检类 message 里明确"仅在异常时输出"，保证投递内容的信噪比；
- 需要分钟级高频轮询时，考虑外部脚本打 webhook，而不是把心跳调到 1m——心跳的价值在低频、便宜、有事才开口；
- 心跳跑在主会话、共享对话上下文，这既是优点（能结合上下文判断）也是污染源，重活一律挪到 cron 的独立会话；
- 不想用心跳时，清空或删掉 `HEARTBEAT.md` 即可，不必改网关配置。

## 总结

cron 和 heartbeat 是互补而非替代：cron 负责确定性调度，heartbeat 负责"低频睁眼看看世界"。我目前的分工是——日报、备份、提醒全走 cron，`HEARTBEAT.md` 只留两三项环境健康检查。调整之后 token 消耗明显下降，群里也不再被无意义的定时消息打扰。选型对了，这两个机制都很省心。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/bd2545accfdca342.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/b89aac563f274d5b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/cd887104e3beb743.png)

