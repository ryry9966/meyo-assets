---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 36658
source: 综合讨论
publishedAt: 2026-09-08
---

## 背景

在 OpenClaw 里做定时自动化，有两条路：**cron**（计划任务）和 **heartbeat**(心跳唤醒)。两者都能让 agent 周期性干活，但机制完全不同。混用或选错，轻则 token 烧得快，重则任务漂移、静默漏跑。

## 两者的本质区别

**cron 是时间驱动**。每个 job 定义在 `cron-jobs.json`（或通过 `openclaw cron add` 创建），标准 5 段 cron 表达式 + 一段作为 prompt 的 message。到点触发，独立于主会话，结果可 announce 到指定频道。适合“每周一 9 点发周报草稿”这类确定性任务。

**heartbeat 是状态驱动**。agent 按 `HEARTBEAT_INTERVAL_MS`（默认 30 分钟）被唤醒，读取工作区的 `HEARTBEAT.md`：文件为空就跳过，有内容才交给 agent 结合上下文判断处理。它本质是一个低消耗的哨兵循环。

## 怎么选：一个判定标准

- 时间点固定、动作明确、一次性或低频 → **cron**。例：每日早报、定时备份提醒、倒计时提醒。
- 周期巡检、做不做要看了状态才知道 → **heartbeat**。例：检查收件箱有无值得提醒的邮件、盯长任务进度、巡检服务状态。
- 秒级/分钟级高频轮询 → 两个都别用。写外部脚本 + webhook/插件推给 agent 更省。

## 做法（步骤）

1. 盘点任务清单，按上面的标准分成两列。
2. cron 任务用 `openclaw cron add` 创建，message 当 prompt 写：写清输入、动作、输出投到哪个频道；一次性任务记得开启跑后自动删除。
3. heartbeat：把周期巡检项写进 `HEARTBEAT.md`，每条只写“检查什么、满足什么条件才动作”；动作细节沉淀到 skill 或引用文档，别全堆在心跳文件里。
4. 上线前手动各触发一次，对照 cron 的 run 记录和 heartbeat 唤醒日志，确认投递目标无误。

## 踩坑点

- **时区**：容器里默认 UTC，`0 9 * * *` 实际在北京时间 17 点跑。要么改容器时区，要么换算好表达式。
- **cron message 写得太笼统**：到点后 agent 自由发挥，产出不可控。把 message 当 system prompt 写，不是一句话备注。
- **heartbeat 间隔贪短**：默认 30 分钟是有道理的，改成 5 分钟大概率变成 token 无底洞。真要高频监控，走外部脚本。
- **把 HEARTBEAT.md 当任务队列**：堆十几个检查项，每次唤醒全量处理，还会和主会话上下文互相污染。控制在 3-5 个轻量检查项。
- **精确时间提醒放 heartbeat**：一定会漂移，这类必须走 cron。

## 可复用建议

- 一句话记住：**“到点做事”用 cron，“有事才做”用 heartbeat。**
- 同周期的多个巡检项合并进 heartbeat，一次唤醒省一次调用；不同周期、需要精准投递的，拆成多个 cron job。
- cron 的 message 进版本库管理，定时任务也是代码，改了要能回溯。
- 每周看一次 cron run 历史，失败后静默的任务最危险。

## 总结

cron 和 heartbeat 不是竞争关系，而是互补的两层：cron 负责“时刻表”，heartbeat 负责“巡检哨”。选型时只需要问自己一句——这个任务是由**时间**触发，还是由**状态**触发？想清楚这一点，基本不会选错。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/0cd5b4f69e671ce7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/9f5c2427e1192588.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/5fc3d7cfd2f0a0dd.png)

