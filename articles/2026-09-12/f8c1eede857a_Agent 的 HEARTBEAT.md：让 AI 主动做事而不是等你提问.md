---
title: Agent 的 HEARTBEAT.md：让 AI 主动做事而不是等你提问
feedId: 37249
source: 综合讨论
publishedAt: 2026-09-12
---

## 背景

OpenClaw 里的 Agent 默认是被动模型：你发消息，它响应。但很多事不需要你发起——每天看一眼磁盘水位、确认某个服务还活着、到点提醒续费域名。为此 OpenClaw 内置了 heartbeat 机制：网关按固定间隔（默认 30 分钟）唤醒一次 Agent，Agent 醒来后读取工作区里的 `HEARTBEAT.md`，按文件内容决定是干活还是继续睡。

## 问题

没有这套机制时，常见方案是外部 crontab + webhook 打进会话。能用，但状态散落：任务定义在系统 cron 里，不在 Agent 的工作区，模型看不到全貌，也无法根据执行结果自我调整。heartbeat 的思路是反过来的——"什么时候想"交给固定间隔，"想什么"交给一个模型随时可读可改的 Markdown 文件。

## 做法

1. 在工作区（默认 `~/.openclaw/workspace/`）新建 `HEARTBEAT.md`；
2. 用自然语言写清任务，例如：

```markdown
# Heartbeat

## 常规检查
- 每次醒来检查磁盘占用，超过 85% 才通知我，否则静默。
- 检查 ~/logs/agent.log 当天有没有 ERROR，有则汇总前 5 条。

## 一次性
- 2025-07-01 后删除本条：周三前续费域名。
```

3. 文件留空或只写 `idle` 时，Agent 回 `HEARTBEAT_OK` 且不打扰你。这是设计关键：**心跳常开，输出可静默**；
4. 用 `HEARTBEAT_INTERVAL` 调频率（如 `15m`、`1h`），设 `0` 关闭；
5. 多 Agent 场景可在文件内按 Agent 分段，各取所需。

调试建议：先把间隔调到 `5m`，加一条"将本次心跳时间追加到 heartbeat.log"，确认链路通了再放宽。

## 踩坑点

- **成本**：每次心跳都是真实的模型调用，空文件也有开销，只是不回消息。间隔别低于实际需要。
- **任务写太模糊**："看看有什么值得关注的"会让模型即兴发挥，动作不可预测。务必写成可判定的条件加明确动作。
- **长任务阻塞**：心跳会话里跑长命令会拖住后续心跳，重活交给异步工具或 cron 类插件。
- **连续失败会熔断**：心跳多次失败后可能被停用，别把脆弱依赖放进高频任务。
- **改动非即时生效**：编辑 `HEARTBEAT.md` 后要等下一次心跳才被读取。

## 可复用建议

- 把 `HEARTBEAT.md` 纳入 git，每次调整有 diff 可查；
- 任务遵循三段式：**触发条件 → 动作 → 汇报策略**（通知 / 静默 / 写日志）；
- 要求 Agent 把心跳结果追加到日志文件，事后可审计它实际做了什么；
- 默认静默、异常才说话，是防止 Agent 刷屏的唯一可靠原则；
- 精确定时（每天 9:00 整）用 cron 工具，heartbeat 只承担"周期巡检"。

## 总结

HEARTBEAT.md 的价值不在"自动执行"，而在把 Agent 的主动性收敛到一个普通 Markdown 文件里：模型能读、能写、你能版本管理和审计。实践上先放宽间隔、把任务写窄，确认行为稳定后再逐步放开。目标是让它安静地值班，而不是兴奋地表演。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/9dcd67ddfb051e4f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/f1371614384892bf.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/74253df064c6bdca.png)

