---
title: OpenClaw 定时任务选型：cron 管“日程”，heartbeat 管“巡逻”
feedId: 36394
source: 综合讨论
publishedAt: 2026-09-07
---

## 背景

OpenClaw 里有两套让 agent 定时干活的机制：**cron**（精确时间触发，到点执行一段固定 prompt）和 **heartbeat**（默认每 30 分钟一次心跳，agent 醒来读 workspace 里的 `HEARTBEAT.md`，自己判断有没有值得做的事）。

两者看起来都像“定时器”，但设计意图完全不同。混用是新手最常见的成本黑洞和故障来源。

## 问题

选错机制的三个典型症状：

1. 用 heartbeat 做“每天 8 点早报”，结果 8:07、8:30 才响——心跳有间隔抖动，还受 activeHours 限制；
2. 用 cron 做“有变化才提醒”的监控，结果什么都发，渠道被无意义消息刷屏；
3. `HEARTBEAT.md` 越写越长，每 30 分钟全量 prompt 跑一遍大模型，月底账单感人。

## 怎么选：先分清任务性质

- 时间点必须准、prompt 固定 → **cron**
- “有变化才说话”的状态巡检，希望 agent 自己判断 → **heartbeat**
- 高频 + 轻量检查（5~15 分钟级）→ heartbeat，且换小模型
- 低频或一次性（某天某时提醒一次）→ cron 一次性任务

另一个隐藏差异：cron 的 isolated 会话每次全新上下文，便宜、可控、没记忆；heartbeat 默认走主会话，agent 更“知情”，但每拍都更贵。**用记忆换判断力，还是用隔离换确定性，是选型的核心权衡。**

## 做法

**cron：管“日程”**

```bash
openclaw cron add \
  --name morning-digest \
  --cron "0 8 * * *" \
  --message "汇总我的日历和未读消息，生成早报发给我" \
  --session isolated
openclaw cron list   # 定期清理过期任务
```

isolated 会话没有聊天记忆，任务依赖的背景写进 workspace 文件或 prompt 本身。具体 flag 以 `openclaw cron add --help` 输出为准，各版本有差异。

**heartbeat：管“巡逻”**

1. workspace 根目录建 `HEARTBEAT.md`，只放“值得醒来检查”的事，条件写具体：

```markdown
- 检查关注仓库有无新 release，有才通知
- 天气 API 预报有雨时，提醒带伞
- 其余情况一律不发消息
```

2. 配置里调间隔、投递目标和 activeHours（字段名以你的版本 docs 为准）：间隔从 60m 起步，target 用 `last` 或 `none`，activeHours 排除深夜。
3. 给 heartbeat 配便宜模型，重活交给 cron 或手动触发。
4. 利用 `HEARTBEAT_OK`：agent 判断无事时会返回该标记且不投递到渠道。如果渠道还是经常收到“无事发生”，说明不触发条件没写清。

## 踩坑点

- **时区**：cron 表达式按网关服务器时区解释。海外 VPS 部署先跑 `date` 确认，别想当然。
- **token 成本**：heartbeat 成本 ≈ 频率 × prompt 长度。`HEARTBEAT.md` 控制在几行内，固定时间的任务全部挪去 cron。
- **孤儿上下文**：cron isolated 会话不记得“昨天聊了什么”，需要连贯上下文的任务要么进 workspace 文件，要么用 main 会话并接受成本。
- **长任务阻塞**：别在 heartbeat 里放爬虫或长流程，下一拍会积压。长活拆成 cron 或独立脚本，agent 只负责触发和汇总。
- **网关停机**：cron 错过即错过（多数版本不补跑），关键告警别押在单条 cron 上。

## 可复用建议

- 一句话分工：**cron 管日程，heartbeat 管巡逻**。
- 巡逻项必须带明确的触发条件和不触发条件，压缩模型自由发挥空间。
- 间隔从能接受的最大值起步，有需要再加密，而不是反过来。
- 每月过一次 `cron list`，清掉过期的一次性任务。

## 总结

两者不是替代关系。把“准点执行”交给 cron，把“条件感知”交给 heartbeat，配上小模型和 activeHours，成本和打扰都能压下来。实践中比较稳的路径是：先用 cron 建立确定性，再逐步把“变了才值得说”的任务迁移到 heartbeat。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/72ac6380e455f900.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/d37bdde63b76a386.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/921ca8deca2c1338.png)

