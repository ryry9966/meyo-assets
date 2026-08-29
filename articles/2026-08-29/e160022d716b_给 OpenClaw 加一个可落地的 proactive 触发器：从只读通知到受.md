---
title: 给 OpenClaw 加一个可落地的 proactive 触发器：从只读通知到受控执行
feedId: 35261
source: 综合讨论
publishedAt: 2026-08-29
---

## 背景

大多数 OpenClaw/Agent 工作流还是“你问一句，它动一下”。对话式助手适合交互，但很多运维、监控、信息追踪场景里，价值来自于“事情发生之后，不用等人开口”。比如依赖库发布新版、日志里出现特定错误、某个 API 状态码异常、定时生成晨报。

Proactive 不是把 cron 换成 AI，而是把 Agent 接入事件循环：由事件触发，让模型基于上下文判断要不要做、做什么、做到哪一步。难点不在“能定时跑”，而在怎么跑得稳、可回滚、可审计。

本文给一个在 OpenClaw 体系里可复用的最小 proactive 闭环。

## 问题

直接给 Agent 挂一个定时任务，通常会遇到三类问题：

1. **触发源与执行环境脱节**：cron 只能给一个入口，无法把事件摘要结构化传给 Agent。
2. **权限边界模糊**：proactive 任务没有用户实时确认，一旦模型误判，可能直接改配置、发消息、写文件。
3. **循环与重入**：Agent 本身如果还能触发其他 Agent，或者任务执行时间超过调度间隔，会出现重复执行或自我放大。

所以需要把 proactive 拆成四层：触发源、判断层、执行层、审计层。

## 做法

### 1. 定义触发源，统一事件格式

不要在 cron 里拼自然语言。先定义一个事件 JSON：

```json
{
  "trigger_id": "log_error_500",
  "source": "loki_webhook",
  "timestamp": "2025-01-01T08:00:00Z",
  "payload": {
    "service": "billing",
    "error_rate": 0.08,
    "window_minutes": 5
  }
}
```

触发源可以是 webhook、文件系统事件、消息队列、定时任务。关键是每个事件都有唯一 `trigger_id`，用于幂等。

### 2. 用 MCP 暴露只读工具

在 OpenClaw 里注册一个只读 MCP server：

```yaml
mcp_servers:
  ops_readonly:
    command: ["node", "ops-readonly-mcp.js"]
    env:
      READ_ONLY: "1"
```

工具只保留 `list_recent_errors`、`get_service_status`、`query_metrics`、`search_logs`。写操作不放这里，比如 `restart_service`、`update_config` 放到另一个需要审批的 MCP server，或者干脆不在 proactive Agent 的工具列表里。

### 3. 给 proactive Agent 明确的运行规则

System prompt 里不要写“你是智能助手，请帮助用户”，而是写清楚运行边界：

```text
你是 proactive-ops。每次运行只接收一个事件摘要。
先调用只读工具核实，不要假设事件一定真实。
如果判断需要执行写操作，只输出 plan，不要直接执行。
输出 JSON，包含 decision、evidence、risk_level、suggested_actions。
```

这样 proactive Agent 的默认输出是“建议”，而不是“动作”。

### 4. 使用无头调用触发 Agent

假设 OpenClaw 实例已暴露 Agent 运行入口，可以使用一个薄调度层：

```python
import json
import hashlib
import openclaw_client  # 示例客户端，按实际 API/CLI 替换

def run_proactive(event):
    idem = hashlib.sha256(
        (event["trigger_id"] + event["timestamp"]).encode()
    ).hexdigest()[:12]

    result = openclaw_client.run_agent(
        agent="proactive-ops",
        input=json.dumps(event, ensure_ascii=False),
        idempotency_key=idem,
        timeout=120,
        max_steps=8,
    )
    return result
```

如果不想写客户端，也可以用 HTTP API + `curl`，或 OpenClaw 的任务队列。关键是每次运行都带 `idempotency_key`，并设置超时和步数上限。

### 5. 结果只进审计日志，告警走独立通道

Agent 输出后，先落日志，再决定是否通知用户。不要让 proactive Agent 直接发消息给所有人。可以只把 `risk_level=high` 或 `decision=action_required` 的摘要推到告警通道。

## 踩坑点

1. **重入**：cron 间隔短于 Agent 执行时长，会重复消费同一事件。用 `trigger_id + timestamp` 做幂等键，或在存储层加唯一约束。
2. **自我触发**：如果 proactive Agent 拥有发消息、调 webhook 的工具，它可能触发另一个 proactive Agent，形成循环。建议全部写操作工具从 proactive Agent 的白名单里拿掉。
3. **上下文膨胀**：不要为了“让模型更聪明”把整段日志塞进去。先做采样、截断、结构化提取。一般事件摘要控制在 500-1000 token 内。
4. **误报疲劳**：阈值设置过灵敏，Agent 频繁报告低风险事件，很快没人看。可以先做影子模式，让 Agent 只记录不通知，观察误报率。
5. **权限漂移**：当 Agent 表现得不错，很容易顺手把写权限也给它。保持 proactive 与交互式 Agent 的权限分离，至少初期不要合并。

## 可复用建议

- **先影子模式，后自动执行**：让 proactive Agent 跑 1-2 周，只输出建议和审计报告，不自动执行。等误报率稳定后再开放低风险动作。
- **决策与执行分离**：Agent 只负责判断和生成 plan，执行由外部执行器按白名单做。这样即使模型幻觉，也不会直接造成破坏。
- **结构化输出是底线**：不要只让模型输出自然语言。JSON 才能被监控、统计、告警。
- **给每次运行设上限**：`max_steps`、`timeout`、`max_tool_calls`，防止 Agent 失控消耗资源。
- **监控三个指标**：空转率（跑了但无结论）、误报率（报了但无需处理）、执行失败率。用这三个指标决定是否扩大触发范围。

## 总结

Proactive 能力不是“更主动的聊天”，而是把 Agent 变成一个值班员：有事件时核实现场，有把握时举手，被授权后才动手。落地时先把触发源、只读工具、结构化输出、审计日志这四件事做扎实，再考虑让 Agent 自动执行低风险动作。这样 proactive 才可控，而不是又多了一个需要人看着的黑盒。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/17e44e8b056403ae.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/24bbd42d60d843d5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/61824a2eaf4d02ff.png)

