---
title: 给 OpenClaw 装一条“主动神经”：Proactive Agent 的受控落地
feedId: 35122
source: 综合讨论
publishedAt: 2026-08-29
---

## 背景

大多数 Agent 还是被动响应式：你发指令，它执行。可很多真正有价值的事，比如磁盘水位、证书过期、服务健康检查、依赖更新提醒，都是“最好别等我开口”。于是很自然会想到给 OpenClaw 加定时触发器，让它主动干活。

但 proactive 不是“定个 cron 自动跑”。如果边界没控好，会很快从“主动助手”变成“自动作死”。

## 问题

我在实践里遇到三类问题：

1. **触发与上下文割裂**：定时 run 没有 run_id、没有固定任务卡片，模型容易自己“脑补”当前状态。
2. **动作权限过大**：直接给 shell 或全量 MCP 工具，模型在有 proactive 能力后会扩大动作范围。
3. **没有收敛机制**：相同告警反复发送，或对同一个异常重复执行动作。

所以需要一个更工程化的结构：触发、取证、决策、动作、收敛五步。

## 做法/步骤

下面是一个“磁盘水位主动巡检”的最小实现。假设 OpenClaw 支持 cron/interval 触发，MCP server 提供三个工具：

- `read_disk_usage`：只读，返回挂载点、使用率、可释放空间
- `clean_old_tmp`：动作工具，支持 `dry_run` 参数
- `send_notification`：通知工具

### 1. 固定触发上下文

每次定时 run 开始时，注入固定任务模板，而不是让模型自由发挥：

```yaml
trigger:
  type: interval
  every: 30m
  task: disk-patrol
  inject:
    run_id: "{{run_id}}"
    ts: "{{utc_now}}"
    task_goal: "巡检磁盘使用率，非必要不动作"
```

### 2. MCP 工具分层

只读工具进默认白名单；动作工具必须带 `dry_run`，且默认不执行。

### 3. 取证先行

Prompt 里固定第一步：先调用 `read_disk_usage`，把原始结果放进上下文。禁止在没取证的情况下直接调用 `clean_old_tmp`。

### 4. 阈值与决策输出

让 agent 输出结构化 decision，而不是自然语言：

```json
{
  "evidence": { "usage": 91, "dry_run_reclaimable_gb": 3.2 },
  "should_notify": true,
  "should_execute": false,
  "reason": "清理动作需人工确认，暂不自动执行"
}
```

策略写具体：仅当连续两次采样 `usage > 90%`，且 `dry_run` 可释放 `> 2GB`，才建议 execute；否则只通知。

### 5. 通知落点与人工闭环

`send_notification` 把 run_id、证据、建议动作发到指定群，并附上“确认执行 / 忽略 / 加入自动白名单”。只有加入白名单的节点才允许下一步自动执行。

## 踩坑点

- **重复执行**：定时触发和手动触发并发时，没有锁会重复巡检或重复清理。用 run_id + 状态文件或数据库唯一约束做幂等。
- **工具粒度太粗**：不要把整个 shell 暴露给 proactive agent。按具体动作拆 MCP 工具，能显著降低误操作。
- **时区不一致**：cron 用 UTC，业务指标可能用本地时间，容易误判。统一存 UTC，展示层再转换。
- **通知疲劳**：同类告警没有冷却时间，用户很快会选择屏蔽。建议同类通知 30 分钟内聚合或静默。
- **上下文污染**：主动 run 会往同一个会话里灌入大量巡检输出，污染后续对话。应使用独立 session，并限制工具输出大小。

## 可复用建议

1. 先跑一周“只读+通知”，不接任何动作工具。
2. 所有主动 run 写审计日志：trigger、工具调用、decision、动作结果。
3. 动作工具一律支持 `dry_run`，并在 prompt 里要求先 dry_run。
4. 给动作加冷却时间：同一对象 30 分钟内只动作一次。
5. 用 MCP 的资源/提示词能力，把策略模板和工具边界固定下来，而不是每次靠用户口头约束。
6. 主动通知尽量带 run_id，方便追溯“是哪次触发干的”。

## 总结

proactive 能力不应该是“模型更主动了”，而是“在没人看的时候，模型依然按照你定义的边界做事”。用 OpenClaw 的定时触发、MCP 的分层工具、结构化 decision 和通知落点，可以搭出一个相对稳的最小系统。先让它学会“敲门”，再考虑让它“自己进门”。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/8f4378f4bb609cb7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/6c10348077adc3c4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/f59eb588a89d7c5c.png)

