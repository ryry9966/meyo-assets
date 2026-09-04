---
title: cron 还是 heartbeat？OpenClaw 两种定时任务的选型与踩坑
feedId: 36087
source: 综合讨论
publishedAt: 2026-09-04
---

## 背景

OpenClaw 里能让 agent "主动干活"的机制主要有两个：**cron** 和 **heartbeat**。第一次配置定时任务时很容易纠结：heartbeat 也能周期性触发，为什么还要 cron？反过来把所有事塞进 cron，又会发现 agent 缺上下文，做不了需要判断的任务。这篇帖把两者的定位、配置方法和踩坑讲清楚。

## 两者分别是什么

- **cron**：cron 表达式驱动，到点触发一次独立任务。跑在隔离会话里，带自己的 prompt，跑完可把结果投递到指定渠道。特点：时间精确、行为确定、不占主会话。
- **heartbeat**：按固定间隔（默认 30 分钟）"戳"一下 agent，让它读取工作区的 HEARTBEAT.md，自行判断这轮有没有事要做。特点：跑在主会话里、共享记忆和上下文、有事干活、没事闭嘴。

一句话：cron 是"到点就干"，heartbeat 是"定期巡逻，看着办"。

## 问题：怎么选

我的判断标准很简单：

- 触发时间确定、任务内容确定 → **cron**。例：每天 9 点汇总待办、每周五生成周报、整点提醒。
- 触发时间无所谓，做不做取决于当前状态 → **heartbeat**。例："收件箱有旗标消息就处理"、"磁盘快满就清理"。

## 做法与步骤

**1. 配 cron**

```bash
openclaw cron add \
  --name daily-digest \
  --cron "0 9 * * *" \
  --prompt "读取 ~/notes/todo.md，汇总今日待办并推送" \
  --channel telegram

openclaw cron list   # 查看任务与下次运行时间
```

要点：cron 是隔离会话，prompt 里要把路径、数据来源、投递目标写全，别指望它记得主会话聊过什么。具体参数以 `openclaw cron --help` 为准。

**2. 配 heartbeat**

在 `openclaw.json` 里调整间隔和输出目标：

```json
{
  "agents": {
    "defaults": {
      "heartbeat": { "every": "45m", "target": "none" }
    }
  }
}
```

然后在 HEARTBEAT.md 写清单，一条一件事、动词开头：

```markdown
- 若 inbox/flagged.md 有未处理条目，逐条处理并归档
- 若磁盘占用超过 85%，清理 /tmp 并记录
- 无事则直接结束，不输出任何消息
```

## 踩坑点

1. **时区**。cron 用宿主机时区，Docker 里默认 UTC，`"0 9"` 实际是北京时间 17 点。容器里显式设 `TZ=Asia/Shanghai`，或先跑 `date` 确认。
2. **heartbeat 刷屏**。HEARTBEAT.md 写得含糊，agent 每轮都回一句"没有需要处理的"，消息轰炸渠道。务必写明"无事不输出"，并把 `target` 设为 `none` 或 `last`。
3. **token 成本**。heartbeat 每个 tick 都是一次真实的模型调用，间隔越短越贵。巡逻类任务 30–60 分钟足够，别为了"实时"调到 5 分钟。
4. **重活放进 heartbeat**。巡逻中发现大任务，应该让它转成 cron 任务或留到主会话确认，不要在 tick 里同步跑十几分钟的命令，会卡住主会话。
5. **依赖 daemon 存活**。两者都要求网关在线。笔记本合盖、容器重启都会漏跑；关键任务要么部署在常开机器上，要么接受"错过就错过"。

## 可复用建议

- **两者组合用**：cron 负责重活和定点输出，heartbeat 只做轻量巡检——发现问题先写状态文件，下一轮处理或转交主会话。
- HEARTBEAT.md 控制在 10 行以内，任务要幂等：同一轮跑两次不应产生副作用。
- 定期用 `openclaw cron list` 和网关日志对账，确认任务真的在跑，而不是"配了就不管了"。

## 总结

时间确定用 cron，状态确定才用 heartbeat。cron 给的是确定性，heartbeat 给的是判断力。把"到点要做的事"交给 cron，把"看见才做的事"交给 heartbeat，两条腿走路，自动化才不会既刷屏又漏事。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/26d625cf4244a6db.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/b03f641304576f5b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-04/f86dc88739ece6b9.png)

