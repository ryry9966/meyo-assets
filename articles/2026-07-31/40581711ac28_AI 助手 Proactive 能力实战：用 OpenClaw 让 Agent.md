---
title: AI 助手 Proactive 能力实战：用 OpenClaw 让 Agent 不等你开口就把事办了
feedId: 31105
source: 综合讨论
publishedAt: 2026-07-31
---

## 背景

多数 AI 助手仍停留在“一问一答”的被动模式：你输入指令，它给出回应。但在真实工程环境中，大量工作不需要等人开口——监控报警、定时报告、事件驱动的状态检查，这些场景天然适合让 Agent 主动完成。Proactive（主动式）能力是把 Agent 从“应答工具”提升为“协作者”的关键一步，但工程实现远比演示复杂。

本文以 OpenClaw 框架为底座，结合 MCP 工具与插件机制，给你一个可直接落地的 proactive 方案，重点讲清楚：如何安全地让 Agent 自己决定“什么时候该做事”，以及这件事到底该怎么做。

## 问题拆解

要让 Agent 主动，需要解决三个问题：
1. **触发源**：谁来叫醒 Agent？定时器、事件钩子还是外部消息队列？
2. **决策边界**：Agent 被唤醒后，是否真的需要执行动作？如何防止误触发或幻觉操作？
3. **执行安全**：主动执行的操作会不会带来副作用？失败时如何兜底？

OpenClaw 提供了灵活的插件系统和异步任务能力，很适合做这类扩展。下面以“监控服务器 CPU，当持续高负载时主动分析并通知”为例，给出完整做法。

## 做法与步骤

### 1. 准备 MCP 工具
先用 MCP 协议暴露一个获取服务器 CPU 使用率的工具。工具服务可以很简单，例如返回最近 5 分钟的平均负载：

```python
# mcp_server.py 片段
@server.tool()
def get_cpu_stats() -> dict:
    load = psutil.getloadavg()
    cpu_percent = psutil.cpu_percent(interval=1)
    return {"load_avg": load, "cpu_percent": cpu_percent}
```

将该 MCP Server 接入 OpenClaw 的工具注册表，让 Agent 随时可调用。

### 2. 创建主动触发器（Proactive Trigger）
在 OpenClaw 中新增一个定时任务插件，每 60 秒唤醒一次 Agent。核心逻辑是构造一个特殊的“系统事件”消息，注入到 Agent 的上下文流：

```python
# proactive_plugin.py
from openclaw.plugin import BasePlugin, CronTrigger

class CpuMonitorPlugin(BasePlugin):
    triggers = [CronTrigger(interval=60)]  # 每60秒触发

    async def on_trigger(self, context):
        # 注入一条“系统提醒”，但不直接发给用户
        context.push_event({
            "type": "system:proactive_check",
            "payload": {"task": "check_cpu_health"}
        })
```

这一步关键在于 **不与用户对话通道混淆**，proactive 触发使用独立的消息类型，Agent 可以识别并自行决策。

### 3. 设计 Agent 的决策 System Prompt
在 OpenClaw 的 Agent 配置中，为 proactive 场景单独设置 system prompt 片段，引导 LLM 按逻辑判断：

> “当收到 system:proactive_check 事件且 task=check_cpu_health 时，请调用 get_cpu_stats 工具获取当前 CPU 状态。如果 1 分钟负载持续超过 4.0 且连续 3 次检查均超标，则调用 send_notification 工具向运维频道发送告警分析。否则只记录日志，不执行外部动作。”

这里明确给了决策阈值和连续判断条件，大幅降低“一次性尖刺”导致的误报。

### 4. 执行动作并闭环
如果判断需要通知，Agent 会调用另一个 MCP 工具 `send_notification`，向 IM 或邮件发送消息。消息内容由 LLM 根据 CPU 数据自动生成简短分析（例如哪几个进程占用高、建议检查点）。执行完成后，插件记录本次状态，用于下一次连续判断。

## 踩坑点

### 上下文膨胀
Proactive 触发如果都沿用同一会话上下文，很快会超出 token 限制。**解法**：每次 proactive 任务使用独立、精简的上下文，仅保留必要的工具结果和上一次检查的计数状态，不携带历史对话。

### 循环触发死锁
某些动作（如发送通知）可能又产生新事件，若不加限制会形成死循环。**解法**：对 proactive 动作增加“冷却期”，同一任务类型 5 分钟内只允许执行一次真实外部操作。

### 权限控制
主动执行扩容、重启等高危操作必须走 Human-in-the-loop。**解法**：危险工具一律设置 `require_confirmation=True`，在 OpenClaw 中该标记会将操作挂起，转由预设的审核通道（如管理员确认）通过后才执行。

### 时延与调度精度
CronTrigger 的实际精度受限于 event loop 负载，且异步执行导致两次触发可能重叠。**解法**：添加防重叠锁（简单的 asyncio.Lock），确保同一任务的执行队列串行化。

## 可复用建议

1. **抽象 ProactiveTask 框架**：将触发（Trigger）、决策（Decision）、动作（Action）拆成三个组件，通过声明式配置组合。这样“监控 CPU”“监测新邮件并摘要”等场景可复用同一套逻辑骨架。
2. **利用 OpenClaw 的事件总线**：所有 proactive 触发都走统一事件通道，方便审计和限流。
3. **模拟测试**：在测试环境用 mock 工具返回特定 CPU 数据，验证 Agent 的决策分支是否正确，避免上线后“沉默”或“轰炸”两种极端。

完整的参考实现可在 OpenClaw 社区仓库的 examples/proactive-agent 路径找到（简化版），包含插件模板和决策 prompt 最佳实践。

## 总结

Proactive 能力不是简单的“加个定时器”，而是一整套触发、决策、安全执行的工程组合。在 OpenClaw 的插件 + MCP 架构下，我们可以用事件注入和独立的决策提示，让 Agent 在无人介入时也能可靠地完成检查与轻量操作，同时守住安全底线。这种模式一旦跑通，你会发现 Agent 的价值不再局限于对话窗口，而是真正渗透进业务流程里——不等你开口，就已经把事办了。

---

