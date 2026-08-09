---
title: Subagent 并行编排实战：在 OpenClaw 里让多个 AI 同时干活
feedId: 32250
source: 综合讨论
publishedAt: 2026-08-09
---

## 为什么需要 subagent 并行？

链条式 Agent 调用有一个很难绕开的瓶颈：当你的 pipeline 必须依次请求 CRM、订单中台和物流接口时，整体延迟就是三次 LLM 推理加上三次网络 I/O 的累加。用户等 8 秒和等 20 秒，体验天差地别。

本质上，这三种查询彼此没有依赖关系，完全可以在同一时间片内完成。OpenClaw 的 Agent/MCP/插件体系已经提供了封装良好的工具边界，如果能把每个查询委托给独立的 subagent，并行执行后再聚合结果，就能把总耗时压到最慢那一个的上限附近。

这篇帖子的目标是：给出一个可以直接落地的 subagent 并行编排方案，包括设计、实现、踩过的坑，以及一个可复用的执行器抽象。

## 背景与场景

业务场景很常见——一个智能助手需要回答：“帮我查一下这个订单的详细信息，当前物流状态，以及用户最近三次沟通记录。”

在 OpenClaw 里，我已经有三个 MCP Server 分别封装了：
- 订单查询 MCP（工具 `get_order`）
- 物流跟踪 MCP（工具 `track_shipment`）
- CRM 对话记录 MCP（工具 `list_conversations`）

传统做法是父 Agent 依次调用这三个工具。但如果父 Agent 的单次 `run` 上下文里只能串行解析工具调用，延迟就会累加。

更好的方式：父 Agent 不做具体查询，而是变成一个 **编排器**。它把任务拆成三个子任务，交给三个专用的 subagent，并行执行，然后只负责合并结果。

## 具体做法

### 1. 为每个查询职责创建 subagent

每个 subagent 仅持有完成其任务所必需的工具，以减少上下文杂音，并降低幻觉概率。

```python
order_agent = Agent(
    name="OrderAgent",
    instructions="You only query order details. Return structured JSON.",
    mcp_servers=[order_mcp_server],
    model="gpt-4o-mini",
    response_format={"type": "json_object"},
)

logistics_agent = Agent(
    name="LogisticsAgent",
    instructions="You only track shipment status. Return structured JSON.",
    mcp_servers=[logistics_mcp_server],
    model="gpt-4o-mini",
    response_format={"type": "json_object"},
)

crm_agent = Agent(
    name="CRMAgent",
    instructions="You only list recent conversations. Return structured JSON.",
    mcp_servers=[crm_mcp_server],
    model="gpt-4o-mini",
    response_format={"type": "json_object"},
)
```

关键点：通过 `response_format` 强制 JSON 输出，这是后续可靠解析的前提。

### 2. 在父 Agent 中实现并行调度器

父 Agent 本身不必直接用 `asyncio.gather` 裸调。我在 OpenClaw 里封装了一个 `ParallelRunner` 工具，供父 Agent 调用。该工具输入一个任务描述列表，内部启动 subagent 的异步运行。

简化版核心逻辑：

```python
import asyncio
from typing import List, Dict, Any

class ParallelRunner:
    def __init__(self, max_concurrency: int = 3):
        self.semaphore = asyncio.Semaphore(max_concurrency)

    async def run_subagent(self, agent, query: str) -> Dict[str, Any]:
        async with self.semaphore:
            try:
                result = await asyncio.wait_for(
                    agent.run_async(query),
                    timeout=25.0
                )
                return {"agent": agent.name, "result": result, "error": None}
            except Exception as e:
                return {"agent": agent.name, "result": None, "error": str(e)}

    async def execute(self, tasks: List[tuple]) -> List[Dict[str, Any]]:
        coros = [self.run_subagent(agent, query) for agent, query in tasks]
        results = await asyncio.gather(*coros, return_exceptions=False)
        return results
```

父 Agent 的 system prompt 可以这样写：

> 你是一个订单助手协调器。当用户询问订单综合信息时，你必须调用 `parallel_delegate` 工具，并传入三个子任务描述：查询订单详情、跟踪物流、获取最近沟通记录。收到并行结果后，按用户可读的格式整理回复。

父 Agent 的 tools 列表中注入一个 `parallel_delegate` 工具，其实现内部实例化 `ParallelRunner` 并传入对应的 subagent 与 query。

### 3. 结果合并与错误降级

并行返回的是一个列表，每个元素带 `agent` 来源和 `result` / `error`。父 Agent 在生成最终回复时，必须遵守以下规则：
- 对含有 `error` 的条目显式告知用户“某信息暂时获取失败”，而不是吞掉异常。
- 每个数据点标注来源，防止信息混淆。例如：“【物流状态】 已签收 （来自LogisticsAgent）”。

## 踩坑记录

**并发限流，第一版几乎必炸**

三个 subagent 同时调用同一个模型时，很容易超出 OpenAI / 任意 LLM 提供商的 RPM 限额。一开始没加 `Semaphore`，前几次测试直接撞到 429。解决方案：① 使用信号量限制并发 agent 数；② 在 `run_subagent` 内部实现指数退避重试（`tenacity` 库很好用）；③ 如果预算允许，不同 subagent 用不同的廉价模型（如 `gpt-4o-mini` + `gemini-flash`），进一步分散压力。

**子 Agent 不按格式输出，父 Agent 解析爆炸**

虽然我指定了 `response_format`，但子 Agent 偶发仍然返回纯文本描述。最初的聚合逻辑会因 KeyError 崩溃。后续增加了两层防御：① 在 `run_subagent` 里对返回值做 `try json.loads`，失败时包装成 `{"raw_text": ...}` 并标记 parsing_error；② 父 Agent 的 prompt 中明确教它如何处理 parsing_error 节点——优先展示已知字段，未知部分直接说“数据格式异常”。

**超时引发悬空调用**

起初没设 `wait_for`，一次网络抖动导致某个子 Agent 卡死，整个并行组永远不返回。加上严格的 20-25 秒超时后，超时的子任务返回超时错误，父 Agent 可立即降级。注意，超时后 LLM 调用可能还在远端执行，但不再等待，避免资源浪费。

**上下文隔离被忽略**

subagent 之间是完全独立的运行实例，上下文不会互相污染。但父 Agent 合并时，我踩过一个坑：子 agent 返回的 JSON 中包含工具调用的中间证据，如果直接喂给父 Agent，可能超过上下文窗口。解决：子 Agent 的 prompt 里明确要求“只返回最终答案 JSON，不要包含中间思考过程”。

## 可复用建议

1. **抽象并行执行器为独立模块**：`ParallelRunner` 应当可配置最大并发、超时、重试次数、输出校验函数。不要在每个业务 Agent 里内联并行代码。
2. **子 Agent 输出规范**：所有 subagent 统一返回 `{"status": "ok", "data": {...}}` 或 `{"status": "error", "message": "..."}` 的标准信封，父 Agent 的合并逻辑可以大幅简化。
3. **从低并发开始灰度**：先用 2 个 subagent 验证，确认无竞态、无死锁后再放大。
4. **监控指标**：记录每个子任务耗时、失败率，方便后续判断模型选型和超时阈值。
5. **MCP 连接复用**：如果多个子 agent 使用同一个 MCP Server，注意连接池大小。必要时将 MCP Server 实现为单例共享，否则并发创建连接会触发 MCP 后端限流。

## 总结

Subagent 并行编排不是什么新鲜架构，但在 OpenClaw 的技术栈里落地时，有几个关键工程点：并发控制、结构化输出、超时降级和模块化封装。做好这几步，原本 12-18 秒的综合查询可以稳定压缩到 4-6 秒，对用户体验的提升是立竿见影的。

如果你也在用 OpenClaw 构建多工具智能体，不妨从最简单的并行分解开始——两个子任务也是并行，代码量很小，收益却很实在。

---

