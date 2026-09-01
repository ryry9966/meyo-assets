---
title: cron 还是 heartbeat：OpenClaw 两种定时触发的选型笔记
feedId: 35723
source: 综合讨论
publishedAt: 2026-09-01
---

## 背景

OpenClaw 的 agent 默认是被动响应：消息进来、事件到达，它才醒来。但很多真实需求发生在"没人说话"的时候——早报、定时备份提醒、盯收件箱、巡检工作区文件。为此 OpenClaw 内置了两种定时触发机制：**cron job** 和 **heartbeat**。两者都能让 agent 自主醒来干活，但设计意图完全不同，混用是新手最常见的浪费来源。

## 两种机制的本质

**cron 是闹钟。** 你写死时间表和 prompt，到点把指定 prompt 交给 agent 执行，产出可投递到指定频道。任务是你定义的，agent 只负责执行；session 可选隔离，也可复用主 session。大部分情况下直接在对话里说"每天早上七点把 xx 发给我"，agent 就会用 cron 工具建好任务。

**heartbeat 是巡逻。** Gateway 按固定间隔唤醒主 session，agent 读取工作区的 HEARTBEAT.md，自行判断"这一轮有没有值得做的事"。无事就沉默，有事才说话或动手。判断权在 agent 手里。

一句话区分：**cron 指定它做什么，heartbeat 告诉它该关注什么。**

## 怎么选：三个判断

1. 时间点和产出都固定（日报、定时摘要、每晚清理提醒）→ cron。
2. 做不做取决于运行时状态（收件箱有无新条目、文件是否异常、待办是否到期）→ heartbeat。
3. 成本敏感度：heartbeat 每一轮都是一次真实的主 session 调用，哪怕最后沉默也会消耗 token；cron 隔离 session 只在触发时消耗。

cron 建议隔离 session + 投递，prompt 里写清输入来源和输出格式：

```json
{
  "name": "morning-brief",
  "schedule": "0 7 * * *",
  "session": "isolated",
  "prompt": "读取 notes/daily.md，生成不超过 5 条的要点日报",
  "deliver": { "channel": "telegram", "to": "<chat-id>" }
}
```

（字段名以你所用版本文档为准，这里只表达意图。）

heartbeat 则维护一份克制的 HEARTBEAT.md：

```markdown
# 巡检清单
- inbox.md 有超过 30 分钟未处理的新条目 → 摘要并提醒
- 任务清单里有今天到期的事项 → 提前 1 小时提醒
- 无事发生 → 保持沉默，不要汇报"一切正常"
```

最后一句务必写上，否则 agent 可能每个周期都礼貌地汇报一遍。

## 踩坑点

- **一次性任务写进 HEARTBEAT.md 忘了删**。之后每个心跳周期都白烧一次调用，还可能反复提醒。HEARTBEAT.md 是值班巡检表，不是待办清单，要定期清理。
- **心跳间隔贪密**。一上来就往几分钟调，token 消耗立刻有感。先用默认间隔跑一周，按实际需要再收紧。
- **cron 隔离 session 没有主对话上下文**。"整理我们刚才聊的内容"在隔离 session 里是无效 prompt，必须写明文件路径或内联内容。
- **cron 走主 session 会污染对话历史**。高频日报类任务建议隔离，避免定时输出稀释主 session 的上下文。
- **时区问题**。cron 表达式按 Gateway 所在时区解释，VPS 在 UTC 时，"每天 7 点"实际是北京时间下午 3 点。要么显式指定时区，要么按 UTC 写表。
- **停机不补跑**。Gateway 重启窗口内错过的 cron 触发就错过了，关键任务要有失败感知，别让日报"默默缺席"。

## 可复用建议

- 决策规则一句话：**固定时间 + 固定产出 → cron；开放巡检 + 上下文判断 → heartbeat**。两者可以共存。
- 一个好用的模式：heartbeat 负责"发现该做的事"，再让 agent 自己建 cron 定时任务，形成"巡逻触发闹钟"的两层结构。
- cron 任务统一命名前缀（如 `daily-`、`weekly-`），方便审计和清理；HEARTBEAT.md 里每个条目都问一句"这事值得每个周期回头看吗"，答不上来就挪走。

## 总结

cron 和 heartbeat 不是竞争关系，而是两种触发语义：一个面向确定性调度，一个面向状态驱动的巡检。把确定性的事交给 cron，把"要不要管"的判断交给 heartbeat，同时对 heartbeat 每一轮的 token 成本保持敬畏——多数"我的 OpenClaw 好像在瞎忙"的问题，根源都是这两者的边界没划清。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/2d022c9fb38d56a6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/40da54e927dd919b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/4d96fd8372e0d320.png)

