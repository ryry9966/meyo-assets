---
title: 不等你开口就把事办了：在 OpenClaw 中构建 Proactive Agent 的工程方法
feedId: 32690
source: 综合讨论
publishedAt: 2026-08-12
---

## 背景

日常使用的 AI 助手多数处于“被动应答”模式——必须由人先发出消息，模型才会动作。在开发运维、数据处理、监控告警这些场景里，真正节省人力的不是更聪明的回复，而是 **Agent 能在条件满足时主动把事情做完**：不等你问“服务器现在负载如何”，它已经把异常分析报告发到频道里；你还没打开看板，日报已经生成。

在 OpenClaw / Agent / MCP 生态里，这种 **proactive 能力** 已经具备初步落地条件，但工程细节和权衡远比想象中复杂。本文以 OpenClaw 框架为基础，结合 MCP 协议和外部调度，给出一种可复现的工程实践，并重点说明踩过的坑和可复用建议。

## 问题

实现 proactive Agent 要解决三个核心问题：

1. **触发源**：谁来叫醒 Agent？定时、Webhook、还是 MCP 资源的变更通知？
2. **上下文注入**：Agent 醒来后，如何获得足够的业务上下文去做出正确决策，而不是凭空“幻觉式主动”？
3. **安全边界**：主动执行修改操作（重启服务、合并 PR、修改配置）时的风险控制，以及如何避免“好心办坏事”。

下面围绕这些问题，给出一套在 OpenClaw 中可行的实现方案。

## 做法 / 步骤

### 1. 以 MCP 资源订阅作为原生事件驱动

MCP 协议里服务器可以向客户端推送 `notifications/resources/updated` 通知，表示某个资源发生了变化。如果你的 OpenClaw Agent 以常驻进程（或长期 WebSocket 客户端）形式运行，可以注册这类通知回调，实现“事件驱动”的主动触发。例如：

- 一个文件系统 MCP 服务器，当 `/data/alerts/` 下出现新文件时，通知 Agent。
- 一个 Prometheus MCP 服务器，当某个告警规则触发出 `alert.yaml` 资源变更时，推送更新。

在 Agent 侧，伪代码如下：

```python
# 注册资源更新回调
@mcp_client.on_notification("resources/updated")
async def handle_resource_update(uri: str):
    if not is_monitored_resource(uri):
        return
    context = await build_context_from_resource(uri)
    decision = await agent.decide(context, tools=available_tools)
    if decision.action != "noop":
        await execute_with_safety(decision)
```

### 2. 用外部调度弥补定时与 Webhook 场景

不是所有触发源都适合 MCP 推送。比如“每天早上 9 点生成报告”或“从 GitHub Webhook 触发代码审查”，可以通过一个轻量调度层，将外部事件转成 Agent 可消费的内部任务消息。

做法是用一个 **scheduler sidecar**（可以是 cron + curl，也可以是 Temporal 这类工作流引擎）调用 OpenClaw Agent 的 `/run` 端点，并在请求中注入结构化上下文：

```json
{
  "trigger": "cron",
  "task_id": "daily_report_09:00",
  "context": {
    "data_sources": ["jira", "github"],
    "recipient": "#team-standup"
  }
}
```

Agent 根据 `trigger` 字段和上下文决定要执行哪些工具链。这样做的好处是 Agent 逻辑与调度解耦，未来可以随意更换调度器。

### 3. 主动行为的安全门：人工确认节点

对于可能产生副作用的操作，直接在 Active 模式下自动执行风险太大。我们在 Agent 工具链中插入一个 **approval tool**，该工具不会真正执行操作，而是向指定的审批人（可以是 Slack、飞书、Web 界面）发送确认请求，并阻塞当前任务直到收到授权。

在 OpenClaw 的 tool 定义里，可以这样拆：

- `restart_service_unsafe`：直接重启，仅在调试环境开放。
- `restart_service_with_approval`：发送审批消息，等待回调，超时自动拒绝。

这层抽象让 proactiveness 变成了“主动建议 + 快速审批”，而不是“完全自动”。

### 4. 任务幂等与熔断

定时触发或事件通知可能出现重复发送，Agent 设计必须保证幂等。我们将每一次主动任务都关联一个唯一的 `task_id`，执行前先检查任务去重缓存（Redis），已完成的任务直接跳过。同时，当一个任务在短时间内频繁失败（例如连续 3 次），自动暂停该触发规则并通知维护者，避免“失控 Agent”消耗资源。

## 踩坑点

1. **通知丢失与重复**  
   MCP 的 `resources/updated` 通知不保证可靠投递。对于关键任务，不要仅依赖这一种触发。应结合定时轮询做兜底，例如每隔 5 分钟主动拉取一次资源状态，与记录的上一次 `version` 对比。

2. **Agent 常驻进程的内存与状态管理**  
   长时间运行的 Agent 实例可能累积对话历史或工具调用结果，导致内存膨胀甚至错误注入。必须设置上下文窗口清理策略，并周期性手动重置 Agent 状态（或每次任务使用新的 session）。

3. **“过度主动”问题**  
   没有冷却机制的主动 Agent 可能在一个短时间内连续触发 50 次相同操作。我们强制为每个触发规则设置最小间隔（例如 10 分钟），并在仪表板上展示最近主动操作记录，让用户随时可关闭某个规则。

4. **工具权限的悄然升级**  
   Proactive Agent 能访问大量 MCP 工具，一旦 prompt 被注入或误判，可能以比你预期更高的权限行事。解决办法是始终以最小权限运行 Agent，并在工具层做二次鉴权，而不是依赖“Agent 自觉”。

## 可复用建议

- **先通知，后操作**：第一个 Proactive 实验建议选“信息推送类”任务（生成日报、监控摘要），等稳定后再放开操作类。
- **所有主动行为必须有可观测性**：任务执行日志写入结构化存储（如 Elasticsearch），并配上简单的现况面板，方便排查“为什么 Agent 没干活”或“为什么干了错误的活”。
- **为主动任务设计特性开关**：在 Agent 配置中为每个 proactive rule 设置 `enabled: true/false`，最好能通过运行时 API 动态关闭，无需重启。
- **利用 MCP 的 Sampling 创建人机协作循环**：对于高风险的决策，用 Sampling 服务器唤起人类确认，而不是完全依赖审批工具，这能获得更灵活的交互。
- **任务上下文尽量显式传递**：不要在 prompt 里隐藏信息，所有需要的数据都通过 MCP 资源或工具参数显式获取，避免 Agent 自行“脑补”。

## 总结

Proactive Agent 是让 AI 从工具晋升为协作伙伴的关键一步。在 OpenClaw 体系内，通过 MCP 资源订阅、外部调度、审批工具和幂等设计，我们已经可以在安全边界内实现不少实用的“不等你开口”场景。但必须承认，当前这种能力仍在工程化早期，可靠性和风险控制需要投入大量心力。

下一次有人问你“Agent 能不能不叫就不动”时，你可以把这套方案摆出来——它会动，只是动得非常谨慎。

---

