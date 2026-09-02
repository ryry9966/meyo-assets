---
title: cron vs heartbeat：OpenClaw 两种定时机制的选型笔记
feedId: 35855
source: 综合讨论
publishedAt: 2026-09-02
---

## 背景

OpenClaw 里能让 agent「定时干活」的机制有两种：cron 和 heartbeat。刚上手时我一度把两者混着用，结果要么 token 白烧，要么该提醒的事没提醒。这篇整理一下两者的实际差异、选型方法和踩过的坑。

## 两者差在哪

**Cron：时间驱动。** 你定义「几点、执行什么 prompt」，到点后起一个（通常隔离的）session 跑完、把结果投递到指定渠道，然后结束。特点是准点、隔离、一次性、冷启动无记忆。

**Heartbeat：状态驱动。** 按固定间隔（默认 30 分钟左右，可调）向主 session 发一个心跳 ping。agent 醒来后检查 workspace 里的 `HEARTBEAT.md`：没事就回一个轻量 ACK，有事才干活并主动发消息。特点是带完整上下文、自主判断，但时间不精确。

一句话：cron 回答「几点做这件事」，heartbeat 回答「现在有没有值得做的事」。

## 选型判断

我的分法很简单：

- **有明确触发时间、内容固定 → cron。** 例如每天早 9 点的日程摘要、每周五的数据周报、整点拉一次 API。
- **触发条件依赖外部状态、需要判断 → heartbeat。** 例如盯收件箱有没有高优先级邮件、目录变化后决定要不要整理、零散的「顺便看一眼」类事项。
- **混合模式最常用：** cron 负责重活（生成报告、批量拉数据），把结果写到 workspace 状态文件；heartbeat 只做轻量巡检，发现状态文件有新内容再决定要不要打扰你。

## 配置步骤

**Cron 侧：**

1. 每个 job 只干一件事，prompt 写清输入、动作、输出格式；
2. 设置 schedule，先确认时区——服务器时区和本地不一致是最常见的「为什么没跑」；
3. 明确投递目标（发到哪个会话/渠道），否则跑完你看不到结果；
4. 需要跨次记忆的任务，让 job 把结果写入文件，下次运行先读。

**Heartbeat 侧：**

1. 间隔从 30–60 分钟起步，别一上来就 5 分钟；
2. `HEARTBEAT.md` 每条 = 触发条件 + 动作。写「若 X 未读且来自 Y，则通知我；否则 ACK」，不要写「帮我关注邮件」这种模糊项；
3. 重活不进 heartbeat：跑脚本、批量分析交给 cron 或插件，heartbeat 只做判断和分发。

## 踩坑点

- **把重任务塞进 heartbeat。** 心跳跑长任务会占住主 session，手动对话排队等锁，体验直接劣化。
- **`HEARTBEAT.md` 越写越长。** 每一拍都带着它进上下文，条目多了 token 线性涨，agent 还容易分心。定期清理失效项。
- **Cron job 需要「上一次的结果」但没落盘。** 每次冷启动无记忆，不写文件就只能从头来。
- **模糊条目导致两个极端：** 要么每拍都打扰你，要么永远只 ACK。解法是把「何时行动」写成可判定条件。
- **Cron 和 heartbeat 撞车会排队。** 不影响正确性但有延迟，cron 错开心跳点即可。

## 可复用建议

- 选型口诀：**时间驱动选 cron，状态驱动选 heartbeat；重活 cron，判断 heartbeat。**
- Heartbeat 条目控制在 5 条以内，每条必须有明确触发条件和动作。
- Cron job = 一次运行 = 一个明确交付物，产出落文件，方便复现和审计。
- 「Cron 写状态、heartbeat 读状态」是两者配合最稳的模式。
- 每周看一次 token 消耗，据此调心跳间隔和清单长度。

## 总结

cron 和 heartbeat 不是二选一，而是分工。把「几点做什么」交给 cron，把「要不要现在打扰你」交给 heartbeat，中间用 workspace 文件传状态。建议先用 cron 跑通确定性任务，再逐步把模糊的「帮我盯着」类需求迁到 heartbeat——这是我验证过最省心的路径。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/f8ab4931aaf8db31.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/b9dc05dbc352bf0f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/a7e3217d892a3fc4.png)

