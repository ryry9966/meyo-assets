---
title: Agent 与 API 的握手：OpenClaw 对接外部服务实战
feedId: 31212
source: 综合讨论
publishedAt: 2026-08-01
---

# Agent 与 API 的握手：OpenClaw 对接外部服务实战

## 背景

当 AI Agent 开始承担真实业务职责时，“调用外部服务”就成了绕不开的环节。无论是查询数据库、触发内部微服务，还是拉取第三方天气、金融数据，Agent 都需要安全、稳定地与这些 HTTP/REST 接口打交道。

直接在 Agent 的 prompt 或代码里拼接 `requests.get()` 的做法，随着接入服务增多会迅速腐化：认证逻辑散落各处、错误处理粗糙、工具描述与实际 API 契约脱节。OpenClaw 社区里越来越多的实践表明，将外部服务对接收敛到统一的**工具层**，并利用 MCP 等标准协议，是让 Agent 的“手”伸得稳的关键。

本文围绕 OpenClaw 的工具注册机制和 MCP 客户端能力，聊聊如何把 REST API 变成 Agent 可靠的外部技能，其中踩过的坑和可复用的封装模式都会坦白交代。

## 问题在哪

- **耦合不可控**：在 Agent 主逻辑里写裸 HTTP 调用，换个接口版本就要改 Agent 核心代码，牵一发动全身。
- **密钥管理混乱**：API Key、Token 散落在配置甚至代码中，轮换时很难无感切换。
- **错误无感传播**：外部服务超时、限流，直接将原始异常抛给 Agent，LLM 收到一堆 HTML 或 JSON 报错很容易产生幻觉，甚至中断任务流。
- **工具描述膨胀**：如果一个 REST API 有几十个参数，全写成 tool schema 会吃掉大量上下文 token，影响 Agent 决策效率。

## 工程化对接步骤

这里的实践基于 OpenClaw 的工具基类和 MCP 客户端能力，你可以用类似思路接入任何 RESTful 服务。以一个“订单查询 API”为例。

### 1. 用 Tool 封装 API 调用

不要手写裸请求。在 OpenClaw 中创建一个 `OrderQuery` 工具类：

```python
from openclaw.tools import BaseTool, ToolParameter
import httpx
import os

class OrderQueryTool(BaseTool):
    name = "query_order"
    description = "根据订单ID查询订单状态，返回JSON结构"
    parameters = [
        ToolParameter(name="order_id", type="string", description="订单唯一标识", required=True)
    ]

    async def execute(self, order_id: str) -> dict:
        api_base = os.getenv("ORDER_API_BASE")
        api_key = os.getenv("ORDER_API_KEY")
        async with httpx.AsyncClient(timeout=10.0) as client:
            resp = await client.get(
                f"{api_base}/orders/{order_id}",
                headers={"Authorization": f"Bearer {api_key}"}
            )
            resp.raise_for_status()
            return resp.json()
```

工具内部统一管理超时、鉴权和数据提取，Agent 只需要关心输入输出结构。

### 2. 密钥一律走环境变量，启用热加载

OpenClaw 的 Secrets 管理器支持从环境变量或 vault 后端获取凭证。开发期用 `.env` 文件配合 `dotenv`，生产环境可以让 Operator 通过 Secrets API 下发，Agent 进程无需重启即可刷新 Token。

### 3. 注册到 Agent 并启用工具调用

在 agent 配置文件中声明工具路径，或使用 `openclaw.register_tool(OrderQueryTool())` 直接注入。之后 LLM 在需要时就会准确生成 `tool_calls`，框架会自动执行该异步逻辑并将结果返回给模型。

### 4. 对接 MCP 兼容的外部服务

如果外部服务已经暴露了 MCP 服务器（例如某些数据库、文件系统），OpenClaw 可以直接作为 MCP 客户端接入。只需在 `mcp_servers.json` 中声明：

```json
{
  "mcpServers": {
    "order-mcp": {
      "transport": "sse",
      "url": "https://api.example.com/mcp/orders"
    }
  }
}
```

之后 Agent 可以直接调用 MCP 提供的 `tools/*` 能力，无需再写一层 REST 封装。MCP 协议本身已经规范了工具列表、调用方式和错误返回。

## 踩坑实录

- **超时区分**：HTTP 客户端超时必须区分连接超时（`connect timeout`）和读取超时（`read timeout`）。如果 LLM 正以流式输出时触发工具调用，而工具执行时间过长，会导致 Agent 整体超时断开。我们最终把工具调用超时设为 5 秒，并在外部服务侧启用异步回调模式处理长任务。
- **错误映射不可偷懒**：直接把外部 API 的 500 页面或者 XML 错误体塞给 LLM，Agent 会开始“编故事”。需要封装一个异常转换层，将 `httpx.HTTPStatusError` 翻译成结构化错误 `{"error": "ORDER_NOT_FOUND", "detail": "订单不存在"}`。
- **tool schema 瘦身**：当 API 参数超过 6 个时，建议只暴露高频必要参数；其余用 `extra_params` 字典透明传递。否则工具描述会轻易超过 1k token，严重影响上下文窗口。
- **流式输出中 tool_calls 截断**：早期 OpenClaw 流式适配中，偶尔出现 `finish_reason` 丢失导致 tool call 未被框架捕获。统一升级到最新的消息协议，并关闭 `stream_options` 中的部分特性后可解。
- **MCP 传输的存活检测**：采用 stdio 传输时，子进程可能随外部服务异常而僵死。我们在 MCP 客户端层加了心跳和自动重启逻辑，避免 Agent 调用卡死。

## 可复用建议

- **抽离 BaseAPIWrapper**：把重试、超时、日志、指标采集都收到一个基类里。每个外部服务只需继承并实现 `_request` 方法，然后由 Pydantic 模型校验返回值。
- **工具分组与目录**：按业务域将外部服务工具注册到不同 Agent 角色下，减少无关工具对模型意图的干扰。
- **灰度开关**：每个外部服务工具加一个环境变量控制 `enabled`，方便线上某个接口超时降级时快速关闭，而不必改代码发版。
- **可观测性**：在封装的 HTTP 客户端里埋入 Prometheus 指标 —— 外部调用成功率、P99 延迟、重试次数。当 Agent 决策怪异时，先看工具侧是否已经“亚健康”。

## 总结

Agent 与 API 的握手，不应该是一堆散落的 `requests` 调用，而应当是契约清晰、可观测、能快速恢复的工具层。OpenClaw 通过 Tool 抽象和 MCP 客户端支持，让我们可以像管理微服务一样管理 Agent 的外部能力。把鉴权、容错、瘦身这些工程细节提前处理好，是让 Agent 从 Demo 跑到 Production 的必经之路。

在 OpenClaw-CN 社区里，已经有不少团队围绕“外部服务网关 agent”做了二次封装，欢迎带上你的实践一起讨论，把 Agent 的手握得更稳。

---

