---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 37002
source: 综合讨论
publishedAt: 2026-09-11
---

## 背景

OpenClaw 是常驻的个人 Agent 网关，跑起来之后第一个需求往往就是"让它按时干活"。框架里有两套定时机制：**cron**（到点触发指定 prompt）和 **heartbeat**（按间隔醒来巡检）。两者都能做定时，但语义完全不同，混着用会踩不少坑。这篇整理一下我的选型判断和配置做法，具体命令参数以你手上的版本为准。

## 两者的本质区别

- **heartbeat**：Agent 每隔固定时间（默认 30 分钟）醒来一次，读一遍 workspace 里 HEARTBEAT.md 的常备清单，自己判断"现在有没有值得做的事"。没事就安静返回，不发消息。它回答的是"**要不要做**"。
- **cron**：调度器在精确时间点触发一个预定义 prompt，任务跑在隔离 session，输出可投递到指定频道。到点必跑。它回答的是"**什么时候做**"。

一句话：cron 是确定性触发，heartbeat 是带判断的周期巡检。

## 怎么选

| 场景 | 选哪个 |
|---|---|
| 每天 8:30 发日报/摘要 | cron |
| 某个具体时间点的提醒 | cron（one-shot） |
| "有新邮件/PR 就告诉我" | heartbeat |
| 磁盘、服务、环境巡检 | heartbeat |
| 固定间隔轮询且每次必须执行 | cron（every 间隔） |

三条判断标准：需要精确时间或保证执行 → cron；允许 Agent 自行判断、没事不打扰 → heartbeat；两者都不满足，再考虑组合。

## 做法与步骤

**heartbeat 侧：**
1. 在 HEARTBEAT.md 写常备检查项，控制在 5 条以内，每条写清"什么条件才算有事"。
2. 间隔先保持默认，轻量巡检不必加密。
3. 关键一步：明确要求 Agent 无事时返回 HEARTBEAT_OK、不发消息，否则它会例行汇报。

**cron 侧：**
1. 用 `/cron add` 或 cron 工具建任务，给 cron 表达式或 `every` 间隔。
2. 一次性提醒用 one-shot；常驻任务确认持久化配置，保证重启后还在。
3. delivery 指到目标频道，别让它默默跑完没人看见。

**组合模式（推荐）：** heartbeat 当哨兵——巡检、异常才报警；cron 当闹钟——定点汇总、定点交付。例如 heartbeat 每 30 分钟查一次构建状态，异常才推送；cron 每天 9:00 把前一天的异常汇总成日报。

## 踩坑点

1. **heartbeat 间隔太密**：每次唤醒都是一次完整推理，10 分钟一次一个月的 token 消耗很可观，先从默认值开始。
2. **heartbeat 刷屏**：HEARTBEAT.md 没写"无事不发"，Agent 会例行汇报，很快被你屏蔽，巡检随之失效。
3. **时区**：容器里默认 UTC，cron 表达式写的"早八"实际是下午四点。统一环境 TZ，或在任务里显式指定时区。
4. **cron 任务缺上下文**：任务跑在隔离 session，看不到你的聊天历史。prompt 必须自包含——路径、参数、输出格式、投递目标全写进去。
5. **双机制打架**：同一件事 heartbeat 和 cron 都在跑，结果重复推送。在 HEARTBEAT.md 里明确排除已交给 cron 的事项。
6. **任务没落盘**：未持久化的 cron 任务重启即丢，重要任务建完先确认持久化状态。

## 可复用建议

- 凡是"到点必须发生"的一律 cron；凡是"有事再说"的一律 heartbeat。
- 把 HEARTBEAT.md 当 SLA 写：每条都要有明确的触发条件和不触发的标准。
- cron prompt 模板化：目标、输入、输出格式、投递频道四要素写全。
- 上线后先看一周调度日志再调间隔，别凭感觉加密。

## 总结

cron 和 heartbeat 不是竞争关系，而是分工：**cron 管确定性，heartbeat 管主动性**。把"必须准时"的交给 cron，把"需要判断"的交给 heartbeat，定时任务体系基本不会乱。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/e4462792b24f5067.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/cedfcd786e3069a1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/df393a884d26aca1.png)

