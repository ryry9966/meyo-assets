---
title: OpenClaw 定时任务选型笔记：cron 管外发，heartbeat 管内务
feedId: 35987
source: 综合讨论
publishedAt: 2026-09-04
---

## 背景

OpenClaw 里能"定时叫醒 agent"的机制有两个：**cron** 和 **heartbeat**。新手常见的问题是：要不要两个都配？或者反过来，全塞进 heartbeat 一劳永逸。我在自己的网关上把两种都跑了几个月，踩过一些坑，把选型思路整理成这篇。

先说清楚两者的本质区别：

- **cron 是"到点必跑"**：经典 cron 表达式（或固定间隔），网关到时间就注入 payload。常用 `announce` 模式——开一个隔离会话跑完，把结果 deliver 到 Telegram/WhatsApp 等渠道。时间、输入、输出目标都是你定的。
- **heartbeat 是"定期自检"**：网关每隔 N 分钟（由 `agents.defaults.heartbeat.every` 控制，默认 30 分钟）唤醒主会话的 agent，让它读 workspace 里的 `HEARTBEAT.md`。没活儿就回 `HEARTBEAT_OK`，用户侧完全无感；有活儿才动手。做什么，由 agent 对照文件自己判断。

一句话：**cron 的确定性来自调度器，heartbeat 的灵活性来自 agent 自己**。heartbeat 跑在主会话里，看得见上下文和自我状态；announce 模式的 cron 是隔离会话，干完即走。

## 问题：怎么选

只问一个问题：**这个任务是否需要在固定时刻产出固定结果？**

- 需要 → cron。典型：每天 9 点推日报、每周日备份、定时提醒。
- 不需要，本质是"条件满足才动手"的巡检 → heartbeat。典型：监看某个目录有没有积压文件、检查证书/磁盘是否快到期、整理记忆文件。

我的口号是：**cron 管外发，heartbeat 管内务**。

## 做法

1. 列任务清单，把所有带明确交付目标的迁到 cron：

```bash
openclaw cron add --name "daily-digest" \
  --cron "0 9 * * *" \
  --payload "汇总昨日 #dev 频道的关键讨论，输出不超过 10 条的摘要" \
  --announce --channel telegram --to "<chat-id>"
```

2. 写 payload 的原则是**自包含**：隔离会话看不到主会话上下文，把数据来源、格式、去向全写进去，按给新同事写交接文档的标准写。
3. `HEARTBEAT.md` 每条任务不超过三行，必须写清触发条件和静默条件，例如：

```markdown
- [ ] 检查 workspace/inbox 是否有超过 3 个未处理文件；
      有则归纳成清单发到 Telegram；没有则保持安静。
```

4. heartbeat 间隔从 60 分钟起步，观察一周再调，别一上来就 5 分钟。

## 踩坑点

- **heartbeat 太频繁 = 烧 token**。每次唤醒即使无事也要读文件、走一轮推理，静默并不免费，5 分钟一次一个月就是近万次唤醒。
- **HEARTBEAT.md 写得太模糊**（比如"帮我盯着项目"），agent 每次醒来都会"努力找活干"，产生噪音消息。每条必须有明确触发条件。
- **cron 时区**：表达式按网关宿主机本地时间执行，容器里若是 UTC，"9 点推送"会整体错位。要么改 TZ，要么换算表达式。
- **announce 没有记忆**：别在 payload 里写"继续昨天的"，隔离会话不知道昨天是什么。
- **静默失败**：cron 挂了不打招呼。定期 `openclaw cron runs` 看执行记录，并确认 deliver 目标真能收到消息——收不到等于没跑。
- **任务重复认领**：同一件事既写进 HEARTBEAT.md 又建了 cron，会双重执行。一个任务只属于一个机制。

## 可复用建议

- cron 管外发（有 deliver 目标），heartbeat 管内务（主会话自我维护），别混用。
- 新任务先问"是否必须在固定时刻产出"，答案直接决定机制。
- heartbeat 间隔宁长勿短，60m 起步，能覆盖就不调短。
- `HEARTBEAT.md` 当成活的 checklist：删掉已完成项，每季度审计一次，别让它无限膨胀。
- cron 的 payload 按交接文档标准写，假设 runner 什么都不记得。
- 定期 `openclaw cron list` 清理僵尸任务。

## 总结

cron 和 heartbeat 不是替代关系，而是分工：前者给你确定性，后者给你自主性。把"什么时候做"交给 cron，把"要不要做"交给 heartbeat，你的网关才会既准时、又安静。选型错了不致命，但选对了，省的是 token 和你半夜被噪音消息吵醒的脾气。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/13317f538fdf75af.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/213511c26528ff62.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/01f1de27c5fc64be.png)

