---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 35862
source: 综合讨论
publishedAt: 2026-09-03
---

## 背景

Agent 跑稳定之后，大多数人的第一个进阶需求是：让它自己动起来，而不是每件事都靠人喊。OpenClaw 给了两条路——**cron** 和 **heartbeat**。两者表面上都是"定时干活"，但设计目标完全不同，选错轻则刷屏、重则烧 token 还错过触发时机。

## 问题

先列几个我见过的典型错法：

- 把"每 10 分钟检查一次部署状态"写成 cron，结果每天多出一堆只为回一句"没事"的独立会话；
- 把"每天 9:00 发日报"塞进 `HEARTBEAT.md`，心跳默认 30 分钟一跳，9:00 的日报经常变成 9:17；
- cron 任务跑在 main session 里，输出和日常对话搅在一起，翻历史像考古。

## 两种机制速览

| | cron | heartbeat |
|---|---|---|
| 驱动方式 | 时间驱动，到点触发 | 状态驱动，周期醒来巡检 |
| 触发精度 | 分钟级、准点 | 默认 30 分钟一跳，有漂移 |
| 任务形态 | 每条任务独立表达式 + 独立 prompt | 共享一份 `HEARTBEAT.md` 清单 |
| 会话 | 可选 main / isolated | 通常复用常驻上下文 |
| 擅长 | 准点、日历型任务 | 条件监控、"没事别吵我" |

一句话总结：**cron 回答"几点做什么"，heartbeat 回答"现在有没有需要我处理的"。**

## 怎么配

**cron**：聊天里 `/cron add` 或直接编辑任务文件，表达式 + prompt + 投递目标三件套：

```
/cron add "0 9 * * *" "汇总 ~/logs 里昨天的错误，生成日报发到 daily 频道"
```

长任务建议用 isolated session，输出通过 delivery 指定落点，别让它默认挤进主会话。

**heartbeat**：在 `openclaw.json` 里把间隔调成实际需要的值（比如 `"15m"`），然后在工作区建 `HEARTBEAT.md`，一行一条，只放检查类条目：

```markdown
- 检查 ~/deploy/state.json，若 status 为 failed 则立即通知我
- 每小时查一次即可：磁盘使用率 > 90% 才报警
```

低优先级条目显式写明"静默/仅异常时说话"，避免每个心跳都刷一句 OK。

## 踩坑点

1. **心跳不是定时器**。默认 30 分钟一跳且实际间隔会漂移，任何带"准点"要求的任务都别放心跳。
2. **心跳是按次付费的**。每次醒来都消耗上下文，`HEARTBEAT.md` 越长越贵。条目控制在个位数，重活写成"发现 X 就转交/起一个任务"，而不是在心跳里现场干。
3. **cron 的 session 选择**：main 能读到对话上下文但会插话；isolated 干净但拿不到历史，prompt 里必须把路径、目标频道、判断标准写全。
4. **时区**：容器里多半是 UTC，cron 表达式按网关时区解析，配之前先 `date` 确认。
5. **错过的 cron 不补跑**：网关重启窗口内的任务直接消失。关键任务让 prompt 结尾落盘状态、或加"若上次未执行则补跑"的判断。
6. **心跳沉默策略没配好**，agent 每跳回一句 OK，频道很快废掉。

## 可复用建议

- 默认策略：**时间锚点用 cron，条件监控用 heartbeat**。拿不准就先用心跳起步（便宜、可随时改），需求稳定后再固化成 cron。
- 把 `HEARTBEAT.md` 当"哨兵清单"，把 cron 当"日程表"，两边职责不混。
- 都要可观测：cron 结果落盘到文件，心跳只在异常时发声。
- 每月清一次 cron 列表和心跳条目，下线不再需要的，自动化清单和代码一样会腐化。

## 总结

两者不是替代关系。cron 是日程表，heartbeat 是哨兵。让准点的事走 cron，让"看一眼、没事别吵我"的事走心跳，agent 的自动化才能既准时、又安静、又省钱。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/3e97a401c9d69816.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/2ce419217dcb212a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/8fe6af6703369fb4.png)

