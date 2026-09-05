---
title: OpenClaw cron vs heartbeat：定时任务选型与分工实践
feedId: 36232
source: 综合讨论
publishedAt: 2026-09-05
---

# 背景

OpenClaw 的 gateway 里有两套让 agent"定时动起来"的机制，新人很容易混淆：

- **cron job**：由调度器驱动的独立任务，支持 cron 表达式和固定间隔，可跑在隔离会话里，结果能投递到指定渠道；
- **heartbeat**：主会话的周期性心跳，默认每 30 分钟唤醒一次 agent，让它读一遍 `HEARTBEAT.md`，有事就做，没事回一个 `HEARTBEAT_OK`，静默不打扰。

混用的结果通常有两种：要么错过精确时间点，要么主会话被定时任务塞满无关上下文。

# 问题

选型其实只看两个维度：**时间是否确定**、**是否依赖主会话上下文**。

| 维度 | cron | heartbeat |
|---|---|---|
| 触发 | 表达式/间隔，到点必跑 | 固定间隔唤醒，忙时可能跳过 |
| 会话 | 可隔离，上下文干净 | 固定在主会话 |
| 决策 | 调度器说了算 | agent 自主判断，没事静默 |
| 适合 | 定时报表、备份、定点提醒 | 条件巡检、收尾跟进 |

一句话：**固定时刻用 cron，"有事才说"用 heartbeat**。

# 做法

## 用 cron 做确定性调度

```bash
openclaw cron add \
  --name weekly-report \
  --cron "0 9 * * 1" \
  --tz Asia/Shanghai \
  --session isolated \
  --message "汇总上周 gateway 日志与告警，输出 Markdown 周报" \
  --deliver --channel telegram --to "<chat-id>" --announce
```

要点（具体 flag 以 `openclaw cron add --help` 为准）：

1. `--session isolated` 开隔离会话，跑完即弃，不污染主会话；
2. 不加 `--deliver`，结果只留在本地运行记录里，容易"跑了等于没跑"；
3. 先 `openclaw cron run <jobId>` 手动验证，再启用；历史用 `openclaw cron runs` 追。

## 用 heartbeat 做"有条件"的巡检

两步：

1. 在 workspace 写 `HEARTBEAT.md`，把检查项和动作条件写死；
2. 在配置里调频率和投递目标：

```json
{
  "agents": { "defaults": { "heartbeat": { "every": "30m", "target": "last" } } }
}
```

`HEARTBEAT.md` 示例：

```markdown
- 只在发现有失败超过 3 次的 cron job 时，整理摘要并提醒我
- 检查收件箱是否有待办
- 其余情况一律回复 HEARTBEAT_OK
```

# 踩坑点

- **heartbeat 不精确**：主会话正在跑任务时，这轮心跳会被跳过或顺延。别拿它做"必须 9:00 发出"的事。
- **heartbeat 是持续成本**：每次唤醒都消耗 token，哪怕最后只回一个 OK。把间隔调到 5 分钟当轮询用，账单会很诚实。
- **cron 隔离会话没有记忆**：message 必须自包含——路径、agent id、期望输出格式全写进去，别指望它"记得我们之前聊的"。
- **时区坑**：cron 默认按网关时区算，服务器多为 UTC，务必显式 `--tz`。
- **投递链路要闭环**：没配 `--deliver`、又没人看 `cron runs`，任务失败也无人知晓。

# 可复用建议

1. 决策口诀：**cron 是闹钟，heartbeat 是值班巡视**；两者可叠加——cron 负责定点触发，heartbeat 兜底检查"该做而没做的事"。
2. `HEARTBEAT.md` 保持一屏以内，检查项越少越准，"其余一律 HEARTBEAT_OK"这句一定要写。
3. cron 的 message 按"给一位陌生同事的完整工单"来写，不依赖任何对话记忆。
4. 新任务上线流程：手动 `cron run` 验证 → 观察 `cron runs` 两三天 → 再放开渠道投递。

# 总结

cron 和 heartbeat 不是竞争关系，而是分工：前者解决"几点做"，后者解决"有事才做"。分清这一点，OpenClaw 的定时自动化基本不会再出乱子。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/ea2d0fa33885ec01.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/be71f1a97fc93511.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/867fbf5ddd571b79.png)

