---
title: 让 AI 助手主动“看家”：基于 OpenClaw Agent 与 MCP 的 Proactive 实践
feedId: 31983
source: 综合讨论
publishedAt: 2026-08-07
---

# 让 AI 助手主动“看家”：基于 OpenClaw Agent 与 MCP 的 Proactive 实践

## 背景：从“你问我答”到“不等你开口”

当前大多数基于 LLM 的 AI 助手依然是响应式设计——用户发一条指令，助手处理一次。即便有了 Agent 和工具链，触发权仍握在用户手里。这对运维巡检、线上告警跟进、依赖方数据变更检测等场景并不友好：等你发现再去问，窗口期可能早就过去了。

OpenClaw / Agent / MCP 的技术栈本身就具备“主动执行”的所有拼图，只是社区里关于如何让 Agent 自主巡检并把事办了的务实分享还不多。本文不聊概念，只讲一套可在你本地或内网跑起来的 proactive agent 方案，附踩坑复盘和可复用建议。

## 问题定义

需求很简单：我们有一批内部服务，需要 Agent 每隔 N 分钟检查关键健康指标（Prometheus 指标、DB 慢查询数、消息队列积压量），一旦某指标越过阈值，主动分析原因，并执行预设的处置动作（重启服务、清理队列、降级通知等）。

技术约束：
- 不引入外部 SaaS 调度平台
- 动作执行前必须经过显式的安全校验
- 巡检结果可追责、可回溯
- 上下文不随循环轮次无限膨胀

## 做法：将 Agent 接入一个定时驱动的 MCP 循环

整体思路：用 OpenClaw 的 Agent 框架 + MCP server，架设一条“主动巡检管道”。架构示意如下：

```
          ┌───────────────┐
          │  Cron MCP     │────定时触发────▶ OpenClaw Agent
          └───────────────┘                       │
                                                  │ 调用工具
          ┌───────────────┐                       │
          │ Prom MCP      │◀──查询指标────────────┘
          └───────────────┘
          ┌───────────────┐
          │ Executor MCP  │◀──执行处置────────────┘
          └───────────────┘
          ┌───────────────┐
          │ Notify MCP    │◀──发送结果────────────┘
          └───────────────┘
```

### 1. 构建定时触发器 MCP

使用 MCP 的 `resources` 或 `tools` 实现一个简单的 cron MCP server。每次触发时，向 Agent 注入一条结构化的 `system event`：

```
{"event":"tick","timestamp":"2025-04-09T10:00:00Z","metadata":{}}
```

我们内部用了一个轻量的 Python MCP server，核心逻辑不足 80 行，基于 `schedule` 库，每 5 分钟通过 MCP 协议向 Agent 发送一次通知。

**踩坑点**：不少同学会把“定时”直接坐在 Agent 内部轮询（比如 `while True: sleep(300)`），这会导致单次任务阻塞时整个检查链停摆，且 Agent 异常退出后循环丢失。建议将定时与 Agent 逻辑解耦，MCP server 作为独立进程运行，失败回调一律走日志和告警通道。

### 2. 编写 Proactive Agent 的 system prompt

我们为 Agent 设定了清晰的巡检清单，格式如下（脱敏版）：

```
You are a proactive infrastructure watchdog.
Every time you receive a "tick" event, you MUST:

1. Query Prom MCP for the following metrics:
   - cpu_usage{service="api-gateway"}
   - queue_depth{queue="order_processing"}
   - db_slow_query_count

2. For each metric, apply threshold:
   - cpu_usage > 80% for 3 consecutive ticks → considered "critical"
   - queue_depth > 5000 → "warning"
   - db_slow_query_count > 10 in 5 min → "warning"

3. If any threshold is breached, analyze recent logs via Log MCP to form a hypothesis.

4. Before any fix action, call Executor MCP's "confirm_fix" with a justification. 
   Only proceed after receiving explicit approval.
   For critical restarts, require a manual override via Notify MCP.

5. After completing the tick, permanently forget detailed logs from this tick
   to control context size. Keep only a short summary in a persistent note.
```

Agent 在每次 tick 结束后，主动调用一次 `trim_context` 工具（我们封装的 MCP 内部能力），只保留当前轮次的状态摘要，避免上下文线性膨胀导致 token 消耗爆炸。

### 3. 上下文管理：显式遗忘而非依赖窗口

LLM 上下文越长，推理越慢、成本越高，且 Agent 容易在旧信息里“迷路”。我们的策略是：

- 每个 tick 执行完毕后，Agent 自行总结三点：(1) 本周期触发了什么异常；(2) 执行了什么动作；(3) 下一个 tick 需要重点关注的指标。
- 将总结写入一个专门的 `state` MCP resource，下次 tick 开始时只加载这个 state，不看以往的原始日志。

这个做法比设置固定上下文窗口更可控，而且能让 Agent 在连续多轮运行中仍有“持续关注”的能力。

**踩坑点**：刚开始我们试过完全让 Agent 自主决定遗忘内容，出现过总结遗漏关键异常链路的 case。后来改为结构化字段，强制填写异常状态和关注项，才把漏报率降下来。

### 4. 安全门：分离“分析”与“动作”

Proactive 意味着 Agent 可能在你睡觉时决定重启某台机器。必须设置硬约束。

我们引入了两层安全门：
- **工具层面的 approval**：Executor MCP 在执行危险动作前，检查是否有 `confirmed: true` 标识。Agent 必须先调用 `confirm_fix` 获得 token，才能带上标识调用实际执行工具。
- **全局冷却期**：同一种处置动作（如重启同一服务）在 30 分钟内只能执行一次，由 Executor MCP 内部维护 Redis 计数器。

这些限制不在 LLM 层面实现，而在 MCP server 的代码层，避免了 prompt injection 绕过。

### 5. 处理重复事件和幂等性

当外部监控系统也在推送事件时，同一个异常可能被 Agent 多次处理。我们在 state 中记录了最近 5 个已处理的 event_id，每次 tick 开始前做去重。配合 Executor 层的幂等设计（基于服务器指纹+时间窗口），有效避免了重复重启。

## 可复用建议

- **先做“只读巡检”，再开“写”权限**：让 Agent 只输出分析报告和告警，观察 1-2 周，等脾气摸透了再打开动作许可。
- **限制最大动作次数/天**：在 MCP server 侧做全局计数器，超过阈值自动降级为只读模式并强通知管理员。
- **把巡检逻辑拆成多个小 Agent**：与其造一个全能守夜人，不如按服务拆分，单点故障只影响局部巡检。
- **监控 Agent 本身**：Agent 如果停止 tick 接收超过 2 个周期，需要专项告警，避免“守夜人睡着了”。

## 总结

Proactive 不是魔法，是一套工程化的定时触发 + 上下文管理 + 安全约束 + 事件去重机制。OpenClaw 和 MCP 给了我们灵活动手的能力，但踩坑点全在非功能性设计上。把这套方案拿到你的环境里，可能需要根据自身工具链调整 MCP server 的实现，但大骨架可以直接复用。

当你的 Agent 第一次在凌晨 3 点准确报告了队列堆积并自动执行了清理，你就知道——proactive，真的香。

---

