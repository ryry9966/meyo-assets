---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 36337
source: 综合讨论
publishedAt: 2026-09-06
---

## 背景

OpenClaw 网关常驻跑起来之后，最高频的进阶需求就是"让 agent 定时干活"。框架给了两条路：

- **cron**：经典 cron 表达式驱动的确定性调度，到点唤醒，执行一条你预设的 message；
- **heartbeat**：周期性心跳，agent 每次醒来读一份 HEARTBEAT.md 巡检清单，自己判断"有没有值得做的事"，没事就返回 HEARTBEAT_OK。

两者看起来都能"定时"，但混用和错用的代价差很多。

## 问题

社区里反复出现三类症状：

1. heartbeat 间隔设成 5 分钟，一天近 300 次模型唤醒，大部分轮次什么都没干，token 账单却翻倍；
2. 把"工作日 9 点提醒站会"写进 HEARTBEAT.md，结果提醒在 9:05～9:40 之间随机漂移；
3. cron 任务跑在主 session 里，过程噪音塞满长期上下文，主对话质量肉眼可见地下降。

根因相同：没分清两种机制的适用边界。

## 做法

### cron：适合时间确定的动作

```bash
openclaw cron add \
  --name "weekday-digest" \
  --cron "10 9 * * 1-5" \
  --session isolated \
  --message "汇总过去24小时仓库的 issue/PR 变化，10 行以内发到频道"
```

三个要点：

- **message 必须自包含**。isolated session 看不到主对话历史，目标、输出格式、投递位置要一次写清；
- **设 timeout**，防止失控任务占住 worker；
- **注意时区**。cron 表达式按网关本地时区解析，海外 VPS 上的"9 点"未必是你以为的 9 点。

### heartbeat：适合状态巡检

heartbeat 的输入是 workspace 里的一份 HEARTBEAT.md：

```markdown
- 有 @我 的 GitHub 通知就汇总，没有则忽略
- 磁盘剩余空间低于 20% 时提醒
- 检查是否有僵尸子进程
```

唤醒间隔在配置里调（如 every: "30m"）。每次心跳，agent 逐条检查，有异常才输出，正常时静默返回 HEARTBEAT_OK。

### 判断规则

就三条：

1. **有确定的时间点吗？** 有 → cron。提醒、日报、定时备份都是。
2. **依赖"当前状态"吗？** 依赖 → heartbeat。盯通知、盯资源、盯进程，本质是巡检。
3. **组合使用**：cron 负责产出（到点干活），heartbeat 负责兜底（巡检异常）。

一句话：**cron 管"什么时候做"，heartbeat 管"要不要做"。**

## 踩坑点

- **定时任务混进了心跳清单**。"每次心跳都有输出"基本就是这种，迁去 cron；
- **清单条目过多**。每次心跳全量执行，又慢又贵，只放真正需要持续盯的事；
- **cron 用了主 session**。过程隔离，只回传结果；
- **误以为 cron 会补跑**。网关停机期间错过的任务不会自动追，关键任务配外部监控；
- **heartbeat 间隔拍脑袋定**。从 30 分钟起步，观察实际触发率再收紧，空转也是成本。

## 可复用建议

- 新需求先过三问：确定时间？依赖状态？产出还是巡检？
- 每月做一次清单审计：逐条问 HEARTBEAT.md"它有确定时间吗"，有就迁去 cron，清单越短越好；
- cron 任务统一命名规范（`daily-`、`weekly-` 前缀），半年后你会感谢自己。

## 总结

cron 和 heartbeat 不是竞争关系，而是两种互补的时间观：前者是确定性的日程表，后者是概率性的巡逻队。把"到点必须发生"的事交给 cron，把"值得持续关注"的事交给 heartbeat，再定期审计两边，agent 的自动化才能既准时又省钱。

（示例命令以你当前版本的实际 CLI 为准，思路不变。）

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/02d7ffb99e91c92e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/0fad2a5726594103.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/5df2886881adcf56.png)

