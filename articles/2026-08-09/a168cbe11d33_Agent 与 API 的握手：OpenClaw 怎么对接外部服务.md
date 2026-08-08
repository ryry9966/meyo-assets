---
title: Agent 与 API 的握手：OpenClaw 怎么对接外部服务
feedId: 32195
source: 综合讨论
publishedAt: 2026-08-09
---

## 一、背景：万能 Agent 也需要一双能伸出去的手

LLM 驱动的 Agent 再聪明，也只是一个推理引擎。脱离了真实数据源和外部操作，它能做的事很快就碰到天花板。这就是为什么在工程化 Agent 项目中，工具调用（tool calling）几乎成了标配。

OpenClaw 作为一个面向复杂任务拆解与执行的智能体框架，在模块边界上做得比较干净：它不假设你会用什么服务，而是留出一套统一定义的“工具”接口，让你把外部 API、数据库、文件操作甚至硬件指令都包装成 Agent 能理解的动作。这篇文章不会去讨论花哨的自主决策，而是聚焦于一个极其实在的问题：怎么让 Agent 稳定、可调试地去调用我们已有的 HTTP API 服务，把推理的“想法”真正落实为一次外部系统里的执行。

## 二、真正的问题不在协议，而在接口契约的匹配

对接外部服务时，很多人的第一反应是关注传输协议——是 REST、gRPC 还是 MCP。但实际上，翻车最多的地方并不在协议层，而在“语义层”的错配。

Agent 输出的是自然语言的变体（哪怕经过 function call 格式化），而 API 期望的是严格类型、严格校验的结构化参数。两者的中间地带就是工具定义中的 `parameters` schema。这个 schema 如果不贴近真实的业务约束，Agent 就会生成看似合理但实际不可执行的参数组合，比如把枚举值写错大小写、把必填字段遗漏、传进了不存在的查询条件。

在 OpenClaw 里，自定义工具一般通过装饰器或配置对象声明。以常见的 `@tool` 定义为例，正确的做法不是简单暴露底层 API 的原始参数，而是做一层面向 Agent 的“转译”：把内部枚举值转化为描述、把分页参数固定默认值、把不必要的技术字段隐藏掉。说白了，工具定义就是 Agent 和 API 之间的“合同”，你得替双方都把条款谈清楚。

## 三、实践步骤：从零对接一个外部 HTTP API

假设你有一个内部订单查询服务，开放了 `GET /api/v1/orders`，支持按状态和创建时间范围过滤。现在要让 OpenClaw Agent 能自主帮用户查询订单，并按意图进行后续操作。整体步骤可以归纳为四步。

**1. 明确暴露给 Agent 的接口语义**

不要直接把 `/api/v1/orders` 的所有查询参数原封不动丢给 Agent。限定状态只暴露几个可操作的值，时间范围用自然语言描述（“最近一天”、“最近一周”）并内部换算成精确起止时间。这能大幅降低 Agent 构造错误调用的概率。

**2. 编写工具函数并声明 schema**

在 OpenClaw 项目中创建一个 `order_tools.py`，用框架提供的 `tool` 装饰器包裹异步函数。关键代码形态如下：

```python
from openclaw.tools import tool
import os, httpx

@tool(
    name="query_orders",
    description="根据订单状态和时间范围查询订单列表",
    parameters={
        "type": "object",
        "properties": {
            "status": {
                "type": "string",
                "enum": ["paid", "shipped", "completed"],
                "description": "订单状态，只能是 paid/shipped/completed"
            },
            "time_range": {
                "type": "string",
                "enum": ["today", "week", "month"],
                "description": "时间范围"
            }
        },
        "required": ["status"]
    }
)
async def query_orders(status: str, time_range: str = "week"):
    base_url = os.getenv("ORDER_API_BASE")
    api_key = os.getenv("ORDER_API_KEY")
    headers = {"Authorization": f"Bearer {api_key}"}
    start, end = _calc_time_range(time_range)
    async with httpx.AsyncClient(timeout=10) as client:
        resp = await client.get(
            f"{base_url}/api/v1/orders",
            params={"status": status, "start": start, "end": end},
            headers=headers
        )
        resp.raise_for_status()
        data = resp.json()
    # 只返回 Agent 关心的摘要字段，避免上下文污染
    return {
        "count": len(data["orders"]),
        "orders": [
            {"id": o["id"], "amount": o["amount"], "status": o["status"]}
            for o in data["orders"][:20]  # 硬限制返回条数
        ]
    }
```

**3. 注册工具到 Agent 运行时**

在 Agent 的启动配置里把 `query_orders` 加到工具列表中。OpenClaw 使用工具的动态发现机制，注册后 Agent 在需要查询订单时会自动生成调用。

**4. 控制返回信息的粒度与格式**

这是很容易被忽略的一步。Agent 的上下文窗口很珍贵，API 返回的完整 JSON 可能成百上千行，不仅消耗 token，还会把后续推理带偏。所以函数返回值要做“压缩”：只保留核心字段、对长列表截断、用人类可读的摘要代替原始数据结构。

## 四、踩坑点实录

下面几个问题都是我实际对接过程中反复遇到的，每一个都可能导致 Agent 行为完全失控。

- **参数校验不在工具层而在 API 层**：如果工具函数内部只是“透传”参数给 API，那么当 API 返回 400 错误时，Agent 拿到的只是一个笼统的异常信息，无法自我修正。要在工具函数里提前做一次前端校验，用明确的自然语言返回错误原因（例如“status 只能为 paid/shipped/completed，你提供了 delivering”），Agent 才可能在下一次尝试中纠正。

- **超时与重试策略缺失**：HTTP 调用天生不稳定。工具函数里如果没有设置合理的超时和有限次数重试（比如 2 次，且只对 5xx 重试），Agent 很可能因为一次临时网络抖动就汇报任务失败，用户体感极差。

- **鉴权信息泄露**：绝对不要将 API key 作为工具参数让 Agent 传递，也不要在返回内容中携带 Token。正确的做法是在工具函数内部通过环境变量或外部 vault 注入，Agent 完全无感。

- **长时间运行导致状态错乱**：如果一个 Agent 任务链中多次调用同一个工具，工具内部要保持无状态。任何需要持久化的上下文（比如 cursor 翻页）应该通过 Agent 的记忆组件传递，而不是在工具里保存全局变量。

## 五、可复用建议

经过多个项目的迭代，我提炼出几条可以复用到几乎所有“Agent + API”场景的工程惯例：

1. **模块化 tool 封装**：为每个外部服务建一个独立的 tool 模块，内部统一处理鉴权、超时、错误转换，对外只暴露干净的 tool 函数。
2. **工具响应的“摘要化”**：坚持给 Agent 返回经过裁剪的结构化摘要，字段数、记录数都设置上限。如果 Agent 需要更多细节，再让它调用另一个更细粒度的“详情查询”工具，形成渐进披露。
3. **用 MCP 统一管理外部工具**：如果对接的服务较多，推荐引入 MCP（Model Context Protocol）作为中间层，将 API 封装成 MCP 服务器，OpenClaw 通过 MCP 客户端统一加载。这样可以在不改动 Agent 配置的情况下增删工具，并且集中管理权限和速率限制。
4. **可观测性兜底**：每次工具调用都记录输入参数、状态码、耗时、返回摘要长度，接入你已有的日志系统。一旦 Agent 行为异常，这是最直接的诊断依据。
5. **人为设置“硬护栏”**：对写操作类 API（退款、删除、发送），一定要在工具定义里加入二次确认参数（如 `confirmed: bool`），让 Agent 必须在获得用户显式确认后才可调用，避免幻觉引发的危险操作。

## 六、总结

Agent 与外部 API 的对接，本质上不是一个技术难题，而是一个接口设计题。你把工具定义看成是 Agent 和 API 之间的中间语言，花时间打磨它的 schema 描述、错误反馈和数据压缩策略，后续的调用可靠性会有数量级的提升。

在 OpenClaw 的生态里，无论是自己写工具函数，还是通过 MCP 集中接入一批服务，核心原则始终没变：把复杂性收敛在工具层，让 Agent 在尽可能干净的语义环境中做决策。当你把这条边界守住了，那些真正的自主任务执行能力，才会自然而然地长出来。

---

