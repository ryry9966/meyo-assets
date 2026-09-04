---
title: cron vs heartbeat：OpenClaw 里两种定时机制，别再用混了
feedId: 36118
source: 综合讨论
publishedAt: 2026-09-05
---

## 背景

用 OpenClaw 做个人自动化，跑得稍久一点都会遇到同一个架构问题：所有"定时"需求要么全塞给 heartbeat，要么全建 cron。两者机制不同，选错了轻则白烧 token，重则任务悄悄不跑、没人发现。

先把差异说清楚：

- **cron**：显式任务表，cron 表达式到点触发，payload 写死在任务里，默认跑在**隔离会话**（isolated session），适合"几点几分做什么"。
- **heartbeat**：全局一个间隔（默认约 30 分钟），每个周期把工作区里 `HEARTBEAT.md` 的检查单投给主会话 agent，由 agent 自己判断这次有没有事要做；没事就回 `HEARTBEAT_OK` 跳过投递。

关键区别就三条：

| 维度 | cron | heartbeat |
|---|---|---|
| 触发依据 | 时间驱动 | 状态驱动（agent 读现场） |
| 会话上下文 | 隔离会话，无主会话记忆 | 主会话，上下文完整 |
| 成本模型 | 只在触发时花钱 | 每个周期至少一次调用，空闲也在烧 |

## 做法与步骤

1. **先分类任务清单**。按"触发依据（时间/状态）、负载（轻/重）、是否需要投递"三列打标，这一步决定归属，比任何配置都重要。
2. **时间确定的进 cron**。示意（字段名以你装的版本为准）：

   ```text
   cron add
     schedule: "0 9 * * 1-5"
     payload:  汇总昨晚 GitHub 通知与日历，输出 ≤200 字早报
     session:  isolated
     deliver:  true → telegram
   ```

   注意 payload 要**自包含**，别写"接着上次的做"。
3. **状态型检查进 heartbeat**。把 `HEARTBEAT.md` 写成短检查单，并要求 agent 维护状态文件做去重：

   ```text
   # Heartbeat 检查单
   - 收件箱有无未处理的重要邮件（对照 state 文件）
   - ~/sync 目录有无未完成任务
   无事返回 HEARTBEAT_OK，不要输出多余内容。
   ```

4. **配置节奏与时段**。heartbeat 间隔别小于 15 分钟，配上活跃时段；跑一周看 token 消耗再调。

## 踩坑点

- **重活放 heartbeat**：跑批、爬虫这类超过一两分钟的任务会占住主会话，后续 heartbeat 排队、间隔变相漂移。重活一律走 cron 隔离会话。
- **cron payload 不自包含**：隔离会话没有历史记忆，模糊指令必然跑偏。路径、参数、输出位置全部写进 payload，或让它去读工作区文件。
- **时区**：容器里默认 UTC，按宿主机时区理解 cron 表达式会差 8 小时，显式设 `TZ`。
- **heartbeat 忘记去重**：没有状态文件时，同一封邮件每个周期提醒一次。让 agent 先读 state、处理完写回。
- **只跑不投**：cron 结果留在会话里没人看，记得配投递目标和渠道；一次性任务跑完及时清理。

## 可复用建议

选型口诀：**几点做 → cron；要不要做 → heartbeat。**

- 预计超过 1 分钟的活 → cron 隔离会话；
- heartbeat 检查单控制在 5 条以内、每条 10 秒内能判断完，超出的转 cron 或子 agent；
- 混合模式最划算：cron 定时产出摘要，heartbeat 只做"有没有异常"的轻量巡检并转发。

## 总结

cron 和 heartbeat 不是替代关系：cron 是调度器，heartbeat 是值班员。把时间确定的任务交给 cron，把"看一眼有没有事"交给 heartbeat，并给它一张短检查单和一个状态文件——成本和可靠性就都在可预期范围内了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/248bef618e81c7e2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/62887a61f8d3630a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/25dc9e33ba48800b.png)

