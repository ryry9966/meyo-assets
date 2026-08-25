---
title: Proactive Agent 最小落地：事件源、dry-run 与权限分层
feedId: 34631
source: 综合讨论
publishedAt: 2026-08-25
---

## 背景：被动 Agent 的边界

目前多数 OpenClaw/Agent 实践仍然是 turn-based：用户发消息，模型思考、调用 MCP 工具、返回结果。MCP 让工具接入更方便，但触发权基本在人手里。巡检、CI 失败、日历冲突、监控报警、依赖安全公告这类场景，理想状态是 Agent 在满足条件时主动发起一次预决策，而不是等你想起它。

## 问题：proactive 的难点不在模型，而在控制

工程上，proactive 能力会立刻撞上四件事：触发什么事件、如何避免重复打扰、写操作如何授权、决策失败如何追溯。很多实践做不下去，不是模型能力不够，而是触发器太噪、权限边界模糊、没有状态去重。

## 做法：一个最小可靠闭环

### 1. 事件源先做窄，不做全

第一版只接一个 cron 或 webhook。例如每 30 分钟拉取一次失败 CI 作业，或监听监控报警。事件进入队列前先预过滤，统一成结构化输入：

```json
{
  "event_id": "ci_fail_20250113_01",
  "source": "github_ci",
  "severity": "high",
  "summary": "main branch e2e failed at install step",
  "occurred_at": "2025-01-13T08:20:00Z"
}
```

不要直接把原始日志塞给模型，token 会爆，判断也不稳定。

### 2. 让 Agent 只做“决策建议”

首版不要自动执行。给 Agent 一个明确协议：根据事件和最近上下文输出 JSON：

```json
{
  "should_act": true,
  "confidence": 0.82,
  "action": "send_message",
  "risk": "low",
  "reason": "same test failed 3 times in 1h"
}
```

只有 `should_act=true` 且 confidence 高于阈值时才进入动作队列。低于阈值的只写日志，不打扰。

### 3. MCP 工具分层

在 MCP server 内把工具分成三类：

- `read_only`：get_ci_status、list_recent_errors
- `reversible_write`：send_message、create_issue、update_status
- `high_risk`：deploy、merge、delete

所有写工具默认带 `dry_run=true`。`high_risk` 必须 `require_approval`。工具描述里明确写出是否有副作用，否则模型容易在 dry-run 下出现误判。

### 4. 状态去重与幂等

用 SQLite 表记录 `event_id`、`decision`、`status`、`action_id`。每次决策前先查 `event_id` 是否已处理。写操作尽量幂等，例如 create_issue 先按 fingerprint 查是否存在相同 issue。

### 5. 反馈与降频

记录用户反馈：“不用管”“太频繁了”可以作为负样本。对同一个 source 设置每日主动动作上限和安静时段。所有主动动作落 `action_log`，保存 prompt hash、工具参数和结果，便于回查。

## 踩坑点

- **上下文污染**：把最近 200 条日志直接丢给模型，结果既贵又不准。先做结构化摘要，事件只给必要字段。
- **重复触发**：没有 `event_id` 去重，同一事件被多次建议，甚至重复创建 issue。状态表必须有。
- **dry-run 被当作真实执行**：工具返回里要明确 `executed=false`，否则后续步骤可能基于假结果继续。
- **权限过大**：Agent 拿到生产写权限时，一次幻觉就足够麻烦。MCP 连接默认只读，写工具单独开。
- **审批卡死**：等待人工确认的任务如果没有超时，会一直占住队列。给审批设 TTL，超时自动关闭并记录。

## 可复用建议

1. 第一个 proactive 场景选“只读巡检 + 每日摘要”，不要一上来就自动修。
2. 决策输出用 JSON Schema 约束，不要自由文本。
3. 所有主动写操作先走 dry-run，再逐步开放低风险写。
4. 事件源、决策、动作三段日志分开，便于定位是触发错、判断错还是执行错。
5. 设置每日主动动作上限和静默时段，控制打扰成本。

## 总结

Proactive 的本质不是让模型“更主动”，而是给 Agent 一个受控触发器、一个明确风险边界、一个可回滚的动作入口。最可靠的第一步，是让它先开口说“我建议处理这个”，而不是直接替你合并 PR。把事件源做窄、把权限收紧、把状态落库，proactive 才会从概念变成可维护的工程能力。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/e4644f8dc3b5234c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/f8f0fe7dbc73d74b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/cc04f5ee63eb4224.png)

