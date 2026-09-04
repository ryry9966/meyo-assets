---
title: OpenClaw 定时任务怎么选：cron 管准点，heartbeat 管兜底
feedId: 36105
source: 综合讨论
publishedAt: 2026-09-05
---

## 背景

OpenClaw 里有两套"定时唤醒"机制，新手很容易混着用。

**heartbeat（心跳）**：网关按固定间隔（`agents.defaults.heartbeat.every`，默认 30 分钟）唤醒主会话，把 workspace 里 `HEARTBEAT.md` 的清单作为轻量 prompt 交给 agent，让它自查"有没有待办"。它的特点是跑在主会话里、共享上下文、精度低。

**cron**：经典调度器。用 cron 表达式定义任务，支持一次性/重复、独立会话、结果投递到指定渠道，还有 `deleteAfterRun`、`wake: next-heartbeat` 这类细节选项。

## 问题

两种典型翻车：全靠心跳，提醒不准点、token 消耗失控；或者把所有琐事都建成 cron，主会话与任务边界混乱。本质问题是没分清需求类型。

## 怎么选：一条判断规则

**需要在某个具体时刻发生 → cron；需要周期性扫一眼 → heartbeat。**

- 每天 7:30 的日报、整点提醒、一次性倒计时 → cron
- 收件箱轮询、待办巡检、备忘兜底 → heartbeat
- 两者叠加最稳：cron 管准点事件，heartbeat 当安全网，接住没有显式计划的事

## 做法

1. **配心跳**，在 `openclaw.json`：

```json
{
  "agents": {
    "defaults": {
      "heartbeat": {
        "every": "45m",
        "activeHours": { "start": "08:00", "end": "23:00" }
      }
    }
  }
}
```

旧版用 `HEARTBEAT_INTERVAL_MS` 环境变量。清单写在 `HEARTBEAT.md`，控制在五行以内。

2. **建 cron 任务**（让 agent 调 cron 工具，或直接 CLI）：

```bash
openclaw cron add daily-report \
  --cron "30 7 * * *" \
  --session isolated \
  --message "汇总昨日 commits 与待办，输出不超过 10 行" \
  --deliver --channel telegram
```

一次性任务加 `--delete-after-run`；轻量任务可以把 wake 模式设为 `next-heartbeat`，搭下一次心跳的车执行，省掉一次独立会话开销。

3. **审计**：`openclaw cron list` 看任务，`openclaw cron runs` 看执行记录；心跳看主会话日志。

## 踩坑点

1. **心跳刷屏**：清单太长，agent 每次都长篇汇报。约定"没事回复 HEARTBEAT_OK 即静默"，间隔别低于 30 分钟。
2. **cron 时区**：Docker 部署默认 UTC，你以为的 7:30 实际是北京时间 15:30。建任务前先在网关里 `date` 确认，或直接给容器设 `TZ`。
3. **isolated 会话没上下文**：cron 默认独立会话，payload 必须自包含。别写"照旧汇总"，要写清数据来源和输出格式。
4. **重复调度**：同一件事既进了 `HEARTBEAT.md` 又建了 cron，会双份通知。改一处记得删另一处。
5. **心跳不是精确定时**：`every: 45m` 意思是"至多约 45 分钟"，负载高时还会顺延，别拿它做整点提醒。

## 可复用建议

- 把 `HEARTBEAT.md` 当配置文件管理：进 git，每次只做行级增删，改动可追溯。
- cron 任务名统一前缀（`daily-` / `weekly-`），方便 list 过滤和批量清理。
- 重活（爬取、汇总）用 isolated + 超时限制；确实需要对话上下文的才用 main 会话。
- 心跳间隔从 30–60 分钟起步，观察一周 token 用量再调，别一上来就 5 分钟。

## 总结

heartbeat 是巡更，cron 是闹钟。准点、一次性、要投递的交给 cron；周期性、低精度、兜底式的交给 heartbeat。职责分清之后，两套机制互相不打架，定时自动化才算真正省心。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/d250e9ed9d0f38ef.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/e4327d679f034fe5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/395875df246e9e40.png)

