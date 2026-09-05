---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 36252
source: 综合讨论
publishedAt: 2026-09-06
---

# OpenClaw 的 cron vs heartbeat：两种定时任务怎么选

用 OpenClaw 挂自动化，绕不开两个入口：cron 和 heartbeat。这帖把两者的分工、配置方式和踩过的坑整理一下，供选型时参考。

## 背景

OpenClaw 的 agent 默认是被动响应的：有消息才动。但很多真实需求是时间驱动的——早报、定期整理、到点提醒、例行巡检。框架为此留了两条路径：**cron** 是独立的定时任务系统，每个任务有自己的调度、会话和投递配置；**heartbeat** 是 agent 级的周期唤醒，到点把心跳消息注入会话，让 agent 自己判断有没有事要办。

## 问题

常见的误用有两类：

1. 把所有定时需求都塞进 heartbeat，间隔越调越密，token 消耗翻倍，主会话历史里混满心跳噪音；
2. 把"让 agent 随时留意收件箱"这类模糊需求硬写成 cron 任务，结果执行时 agent 要么没事干，要么每次都强行汇报。

根因是没分清两种机制代表什么：

| 维度 | cron | heartbeat |
|---|---|---|
| 触发 | 5 字段 cron 表达式，分钟级 | 固定间隔（默认 30m） |
| 会话 | 默认隔离会话 | 主会话，共享上下文 |
| 投递 | 按任务配置（announce / none / webhook） | 无事回复 HEARTBEAT_OK，静默 |
| 数量 | 可建多个，支持一次性任务 | 每个 agent 一个节奏 |
| 适合 | 确定的时间表、日报、提醒 | 巡检、待办检查、被动观察 |

## 做法 / 步骤

**1. 给需求分类。** 列出所有定时需求，分两堆：能写成"几点几分执行"的进 cron；只能描述成"每隔一段时间看一眼、有事再说"的进 heartbeat。这一步能解决八成的纠结。

**2. 配置 heartbeat。** 在 `openclaw.json` 里：

```json5
agents: {
  defaults: {
    heartbeat: {
      every: "30m",
      activeHours: { start: "08:00", end: "23:00" }  // 夜间静默
    }
  }
}
```

然后在工作区写 `HEARTBEAT.md`，内容就是巡检清单：日历有无临近事项、收件箱有无未处理消息、有无过期任务。并明确指示：无事可报时回复 `HEARTBEAT_OK`。这份文件越短越省钱。

**3. 建 cron 任务。** CLI（以你版本为准）：

```bash
openclaw cron add \
  --name morning-digest \
  --cron "30 8 * * *" \
  --session isolated \
  --message "读取昨天的会话摘要和待办文件，生成日报发给我" \
  --delivery announce --channel telegram --to <chat-id>
```

**4. 验证后再挂正式时间。** 先手动触发一次，确认投递渠道和输出符合预期，再观察一周 token 消耗。

## 踩坑点

- **心跳不是免费的。** 每次心跳都是一次模型调用，哪怕只回一句 `HEARTBEAT_OK`。别把间隔调到一两分钟，默认 30m 对多数人够用。
- **HEARTBEAT.md 别写成文档库。** 它每次唤醒都会注入，三行清单和三千字的开销完全不同。长资料放文件里让 agent 按需读。
- **隔离会话没有主会话记忆。** cron 任务的 message 必须自包含：路径、渠道、要读的文件都写清楚。需要跨次运行的状态就落盘到工作区文件。
- **时区。** cron 按宿主机时区解析，服务器是 UTC 时"早八点半"会变成晚上。建任务时显式指定 timezone，或把容器 `TZ` 设对。
- **投递没配等于白跑。** `delivery: none` 适合无人值守的后台整理；要结果就配 announce + 渠道 + 目标，并确认凭证就绪。
- **心跳会被长任务挤掉。** agent 忙时心跳会被跳过，重要定时动作别依赖它。

## 可复用建议

- 一句话判断标准：能写出 cron 表达式的用 cron；只能写成"每隔 N 分钟看一眼"的用 heartbeat。
- 心跳保持便宜，cron 保持自包含。
- 一次性提醒（"明早九点叫我"）用 cron 的一次性任务，跑完即清理。
- 定期 `cron list` 清理僵尸任务。

## 总结

cron 和 heartbeat 不是二选一，而是分工：cron 负责**时间驱动的确定性动作**，heartbeat 负责 **agent 的自主巡检**。先分类需求，再各就各位，配合自包含的任务描述和克制的 HEARTBEAT.md，两套机制可以长期共存、互不干扰。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/f35d208476f02eb0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/d6a4365b2cc82dc6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/8e7ae32f7babdff1.png)

