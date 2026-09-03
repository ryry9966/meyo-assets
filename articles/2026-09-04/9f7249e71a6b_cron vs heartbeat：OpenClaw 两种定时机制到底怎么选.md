---
title: cron vs heartbeat：OpenClaw 两种定时机制到底怎么选
feedId: 35981
source: 综合讨论
publishedAt: 2026-09-04
---

## 背景

在 OpenClaw 里做"周期性触发"，无非两条路：**heartbeat（心跳）** 和 **cron（计划任务）**。我早期踩过典型的坑：什么都往 HEARTBEAT.md 里塞，一个月下来 token 消耗吓人；后来矫枉过正全用 cron，结果任务全成了失忆的孤岛。这篇把两者的边界和我的现行分工方式整理一下。

## 两条机制差在哪

**heartbeat 跑在主会话里。** 每隔一段时间（默认 30 分钟），gateway 给主会话注入一次心跳 prompt，agent 读工作区的 `HEARTBEAT.md`，有事就做，没事回一个简短确认（默认不会转发给你）。它的核心优势是**带上下文**：你上周让它盯的仓库、没聊完的草稿，它都记得。

**cron 是隔离运行。** 每个 job 用自己的 prompt 起独立会话，默认看不到主会话历史；支持 cron 表达式、固定间隔和一次性（`--at`）三种调度，运行记录按 job 可查，结果可投递到指定频道。适合"到点就干、干完就走"的任务。

一句话：heartbeat 是带记忆的巡逻，cron 是带工单的定时工。

## 我的分工做法

1. **模糊的、需要感知的事放 heartbeat**。HEARTBEAT.md 只写可判定的轻量检查项，比如"留意 drafts/ 目录有没有新文件""未完成草稿超过三天就提醒我"。条目我控制在 5 条以内。
2. **有明确时刻表的事放 cron**。早报、周总结、一次性提醒。prompt 必须自包含：要用的数据写明工作区路径让它自己读，不指望它记得任何聊天内容。
3. cron 结果统一投到一个低优先级频道，需要人工介入才 @ 我。
4. 心跳频率按需调，别照抄默认值：

```jsonc
// ~/.openclaw/openclaw.json
{
  "agents": {
    "defaults": {
      "heartbeat": {
        "every": "1h",
        "activeHours": { "start": "08:00", "end": "23:00" }
      }
    }
  }
}
```

（字段名随版本略有差异，以你版本的文档为准。）

## 踩坑点

- **HEARTBEAT.md 条目堆太多**。每次心跳都是一次真实的模型调用，条目要短、要能一句"是/否"判定，否则心跳比正经任务还贵。
- **cron prompt 写了"我们之前聊过的"**。隔离会话里 agent 一脸懵。要么把关键信息写进 prompt，要么落成工作区文件并在 prompt 里给路径。
- **时区**。cron 按 gateway 所在机器的时区跑，容器里往往是 UTC，"早上 9 点"变成下午 5 点。建 job 时显式指定时区，或先 `date` 确认机器时区。
- **定时提醒别靠 heartbeat**。它没有"几点几分"的概念，只有固定间隔，一次性提醒老老实实用 cron 的 `--at`。
- **两个机制都依赖 gateway 进程**。客户端关了没关系，gateway 挂了全停，排查"任务没跑"先看进程。

## 可复用建议

选型口诀：

- 有固定时刻表、结果独立、prompt 一句话能说清 → **cron**
- 需要会话上下文、频率宽松、检查项可枚举 → **heartbeat**
- 拿不准就先写进 HEARTBEAT.md 观察一周，发现高频化、固定化了再迁到 cron——迁移成本很低，反过来不好走。

## 总结

heartbeat 和 cron 不是替代关系，是分工：heartbeat 管常态感知，cron 管精确调度。控制心跳成本的关键在 HEARTBEAT.md 的条目质量，cron 的可靠性靠 prompt 自包含加显式时区。把这两点做对，OpenClaw 的定时任务基本不会再出幺蛾子。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/861303402e3c4cd9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/89ca69491f9b6ca0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/5514053f0ebdc93f.png)

