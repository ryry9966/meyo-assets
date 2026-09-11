---
title: cron vs heartbeat：OpenClaw 两种定时任务怎么选
feedId: 37068
source: 综合讨论
publishedAt: 2026-09-11
---

## 背景

OpenClaw 网关里有两条让 agent "自己动起来"的路：一是 **cron 调度器**，按 cron 表达式在精确时间点触发任务；二是 **heartbeat 心跳机制**，按固定间隔（默认 30 分钟）向主会话注入一次心跳 prompt，由 agent 对照工作区的 `HEARTBEAT.md` 清单决定干活还是沉默——无事时回复 `HEARTBEAT_OK`，消息不会外发到通道。常见误区是把所有定时需求都塞进 heartbeat，结果 token 烧得快、任务还不准时；或者全用 cron，丢掉了"基于上下文主动判断"的能力。

## 问题

两者的本质差异在三点：

- **触发方式**：cron 是"到点必跑"，时间精确；heartbeat 是"定期醒来自己判断"，触发点跟随 tick 间隔，有天然抖动。
- **会话**：cron 支持 isolated 隔离会话（上下文干净、单次成本低），也可指向主会话；heartbeat 固定跑在主会话，带完整记忆和上下文。
- **决策权**：cron 的 prompt 是你提前写死的；heartbeat 每一跳都由 agent 读清单后自行决定做不做。

## 做法

按任务性质二分：

**1. 时间确定的周期/一次性任务 → cron**

```bash
openclaw cron add --name daily-digest \
  --schedule "30 8 * * *" \
  --session isolated \
  --message "拉取昨日 issues 汇总，发到 Telegram"
```

用 `openclaw cron list` 核对任务，`openclaw cron runs --id <id>` 查执行记录。典型场景：每天 08:30 推日报、每周五清理临时文件。

**2. 需要"盯着"的模糊任务 → heartbeat**

- 间隔建议 ≥30 分钟，夜间无人值守时段配置静默；
- `HEARTBEAT.md` 只写 3~5 条检查项，并明确指令："无需要用户知晓的事，只回复 HEARTBEAT_OK"。

**3. 混合模式（推荐）**：heartbeat 当传感器，cron 当执行器。心跳中发现"三天后有个截止日期"，就让 agent 自建一个一次性 cron 任务在准点提醒。判断交给心跳，准时交给 cron。

## 踩坑点

- **时区**：容器里默认 UTC，cron 表达式按网关本地时间解析，先 `date` 确认再写。
- **isolated 会话无记忆**：prompt 必须自包含，需要的上下文要么写全，要么让任务先读状态文件；中间状态建议落盘复用。
- **HEARTBEAT.md 越堆越多**：agent 每跳都要全量读一遍，成本上涨、判断反而变差。超过 5 条，说明该转成 cron 了。
- **心跳间隔调太小是 token 无底洞**：5 分钟一跳，全天大部分跳都是空转计费。
- **长任务别放 heartbeat**：一跳没跑完下一跳又来，行为重叠错乱；重活交给 cron。
- **cron 投递失败偏静默**：先到 `cron runs` 里确认送达状态，再配失败告警，否则任务"看起来在跑，其实没人收到"。

## 可复用建议

- 一句口诀：**准点的用 cron，看情况的用 heartbeat**。
- 每条心跳检查项都要能回答"什么条件下必须打扰用户"，答不清就删掉。
- isolated cron 任务把产物写入固定文件，形成可审计的运行痕迹，也方便下一次运行读取。
- 一次性任务务必带自动删除，并定期 `cron list` 清理僵尸任务。

## 总结

cron 和 heartbeat 不是二选一，而是分工：**cron 负责确定性，heartbeat 负责主动性**。先把"到点要做"的事全部迁到 cron，把心跳清单裁剪到最小，再用"心跳发现 + cron 执行"的组合处理跨期任务，定时体系基本就稳了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/ad2e28ac2855c9dc.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/ec1b5b2b5b84d8e8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/1bc00b0d8bc672d3.png)

