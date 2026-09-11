---
title: cron 与 heartbeat：OpenClaw 定时任务的选型笔记
feedId: 37115
source: 综合讨论
publishedAt: 2026-09-12
---

## 背景

让 Agent「自己动起来」是自动化的第一关。OpenClaw 内置了两条定时路径：**cron** 和 **heartbeat**。社区里常见的问题不是「怎么配」，而是「该用哪个」——两者都能让模型周期性醒来，但设计目标完全不同，选错了轻则浪费 token，重则任务根本跑不对。

## 两者到底差在哪

- **heartbeat**：固定间隔的「值班巡检」。Gateway 按间隔向主会话注入一条心跳提示，Agent 读工作区的 `HEARTBEAT.md` 决定要不要做事，没事就回 `HEARTBEAT_OK`，不产生任何投递。它跑在主会话里，天然共享你的对话上下文。
- **cron**：日历语义的「排班表」。支持标准 cron 表达式、one-shot 和固定间隔，每次执行可选 isolated 隔离会话或 main 主会话，跑完可把结果投递到指定频道。

一句话：heartbeat 回答「现在有没有事要做」，cron 回答「到点了要做某件事」。

## 做法

**1. 时刻明确的任务用 cron**，例如工作日早报：

```bash
openclaw cron add \
  --name "daily-brief" \
  --cron "30 9 * * 1-5" \
  --tz "Asia/Shanghai" \
  --session isolated \
  --message "读取 workspace/notes 里昨天的记录，输出 10 行内简报" \
  --deliver --channel telegram --to "@mychannel"
```

**2. 不确定时刻的巡检用 heartbeat**。在配置里设间隔与活跃时段：

```json
"heartbeat": {
  "every": "30m",
  "activeHours": { "start": "08:00", "end": "23:00" }
}
```

同时在 `HEARTBEAT.md` 写明确的检查项（收件箱、待办文件、某个目录变化），而不是「看看有没有事」。

**3. 上线前手动验证**：`openclaw cron run <jobId> --force` 跑一次，用 `openclaw cron runs <jobId>` 看历史与投递结果。

## 踩坑点

- **时区**：cron 默认跟系统走，海外 VPS 在 UTC，「早 9 点」会错 8 小时，务必加 `--tz`。
- **isolated 会话没有主会话记忆**，message 里要写清去哪读上下文，否则每次执行都「失忆」。
- **heartbeat 间隔太小**（比如 1 分钟）会持续消耗主会话 token，还可能和正在进行的对话互相打断。
- **cron 挂到 main 会话**虽然能共享记忆，但每次运行都会往主上下文里塞内容，长跑几周后上下文会明显变「脏」。
- **投递失败容易被忽略**：频道掉线时 cron 结果只落在本地，记得定期查 `cron runs`，别只看任务列表里 job 还在不在。
- **心跳和 cron 同时写同一份文件**，低概率互踩，建议把写操作错开。

## 可复用建议

- 判断句：**日历说得出来的用 cron，只有「周期」概念的用 heartbeat**。无法表达「每周一早上」是 heartbeat 的硬边界。
- 把 `HEARTBEAT.md` 当值班检查清单维护，每条写「查什么、什么条件下才动作」，无事明确返回 `HEARTBEAT_OK`，这是控制心跳成本最有效的一招。
- 所有定时任务先 `--force` 跑通再挂日程；投递型任务每周扫一眼执行记录。
- 需要共享记忆的定时任务，优先考虑「cron 读工作区文件」而不是「cron 挂 main 会话」，让记忆落在文件里比落在会话里可审计。

## 总结

cron 是排班，heartbeat 是值班。选型的关键不是功能强弱，而是两问：任务有没有确定的时刻？要不要共享主会话记忆？想清楚这两条，大部分「Agent 不主动」或「Agent 太吵」的问题都能归位。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/5d367d95832952aa.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/2e1bf0c81f45e644.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/13da6e9b12d8c097.png)

