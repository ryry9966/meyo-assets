---
title: 给 Agent 加个心跳：用 HEARTBEAT.md 让 AI 按节奏主动巡检
feedId: 34259
source: 综合讨论
publishedAt: 2026-08-23
---

# 给 Agent 加个心跳：用 HEARTBEAT.md 让 AI 按节奏主动巡检

## 背景

OpenClaw 的 agent 接入 MCP 和插件后，能力不差，但多数时候还是“人问一句，它动一下”。真正需要自动化的场景——队列堆积检查、错误日志巡检、缓存清理、备份验证——如果都靠人记着去问，既不稳定也不现实。

## 问题

为什么不能简单加个 cron？因为 cron 只执行固定命令，而 agent 的价值是在多个工具间做判断：先读队列状态，再决定要不要 drain；先看日志级别，再决定是记录还是升级。缺的是一个稳定的“节奏文件”和“触发-读取-决策-执行”的闭环。HEARTBEAT.md 就是干这件事。

## 做法/步骤

1. 在 agent 工作区根目录创建 `HEARTBEAT.md`，建议用 YAML frontmatter 存元信息，正文写检查项和动作。

```markdown
---
cadence: 30m
scope: [data/queue, logs/error.log, cache/]
---

# Heartbeat

## Checks
- [ ] 检查 data/queue/*.json 是否超过 10 个
- [ ] 检查 logs/error.log 最近 1h 是否新增 fatal
- [ ] 检查 cache/ 体积是否超过 2GB

## Actions
- 队列堆积：调用 queue.peek 看前 5 条，再调用 queue.drain --dry-run
- 发现 fatal：提取最近 5 条 fatal 摘要，写入 reports/heartbeat/$(date).md
- 缓存超限：只记录不删除，写入 reports/heartbeat/$(date).md

## Report
- 每次运行追加一行到 heartbeats.log：时间、检查项数、动作数、是否升级
```

2. 配置定时触发。可以用 systemd timer、OpenClaw 的调度插件，或外部 cron 调 CLI/MCP：

```bash
# 伪命令：不同部署方式可能不同
openclaw run --agent heartbeat \
  --prompt "Read HEARTBEAT.md, read state.json, execute due checks, update state.json, write heartbeats.log"
```

关键是每次触发时，不让模型直接改 `HEARTBEAT.md`，而是读它、执行、把运行状态写到单独的 `state.json`。

3. 把有副作用的操作绑定到 MCP 工具，工具描述里写清副作用。比如 `queue.drain` 必须接受 `--dry-run`，`log.scan` 只读。这样模型决策时不会随手删数据。

4. 每次执行后更新 `state.json`，记录 `last_run`、`last_action`、`cursor`、`failure_count`。下次运行先读 state，避免重复动作。

## 踩坑点

- **重复执行**：模型不知道上次跑到哪，可能反复 drain 同一批队列。解决：`state.json` 里存 `cursor` 或 `last_queue_id`，执行前先读，执行后立刻更新。
- **上下文污染**：如果把历史心跳全塞进上下文，几轮后模型会开始“编造”之前的检查结果。解决：只保留最近 3 次 run 的摘要，不喂完整日志。
- **权限过大**：第一周千万别给 agent 写文件的 shell 权限。只读检查 + 报告，写动作必须走 MCP tool，并带 `--dry-run`。
- **打扰频繁**：每 10 分钟发一次“一切正常”很烦。默认只记录不通知，仅异常或升级时发消息。
- **格式被模型改乱**：不要用 `HEARTBEAT.md` 本身存 `last_run`，模型可能乱编辑。用单独 JSON 状态文件，或让工具更新状态。

## 可复用建议

- 从一个 check 开始，别一上来就做自动修复。
- 把 `HEARTBEAT.md` 纳入版本控制，`state.json` 和 `heartbeats.log` 不入库。
- 频率按业务容忍度：巡检类 30m–2h，队列类 5–15m，备份验证类每天一次。
- 给 agent 一个停止规则：同一问题连续 3 次无法解决，就升级人工，不继续自动重试。
- MCP 工具的描述要写清楚副作用和参数边界，这比改模型 prompt 更有效。

## 总结

`HEARTBEAT.md` 不是让 agent 变“自律”的魔法，而是把自主性限制在可预测节奏里的工程化手段。它让 AI 从“等你问”变成“按节奏看、按条件做、按级别报”。如果你的 OpenClaw 实例已经接好了 MCP 和插件，只是缺乏主动触发，这个文件可以作为最小可行起点。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/75a1c23517860bd6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/1e13cf616e5ec588.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/1006a4fd96134e0d.png)

