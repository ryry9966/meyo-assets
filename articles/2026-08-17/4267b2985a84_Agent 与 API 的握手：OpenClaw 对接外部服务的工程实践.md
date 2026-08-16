---
title: Agent 与 API 的握手：OpenClaw 对接外部服务的工程实践
feedId: 33535
source: 综合讨论
publishedAt: 2026-08-17
---

## 背景

OpenClaw 的 Agent 能力再强，也不可能把企业内部系统、第三方 SaaS 的数据都内置进去。实际项目里，Agent 经常需要查询订单、同步客户信息、调用通知服务，或者拉取外部数据源。如果每次都让模型自己拼 HTTP 请求、处理鉴权和重试，不仅容易出错，还会把密钥暴露在上下文里。

更麻烦的是，外部 API 的响应格式、限流策略、错误码五花八门。Agent 直接面对这些细节，往往会“幻觉”出错误参数，或者把一条 500 错误当成业务结果继续执行。

所以我们需要一个稳定的连接层：把外部 API 封装成 OpenClaw 能识别的工具，让 Agent 通过工具调用而不是裸 HTTP 请求来访问外部服务。

## 问题

常见的做法有两种：一是写 OpenClaw 插件，在插件内部调用 API；二是通过 MCP（Model Context Protocol）接入外部服务。从工程角度看，MCP 更适合做这件事：它把“工具”定义和实现解耦，OpenClaw 只需配置加载，不关心你的 API 是用什么语言封装的。

但不管用哪种方式，对接外部 API 都会遇到几个共性问题：

1. **鉴权**：密钥不能进模型上下文，也不能硬编码在配置里。
2. **超时与重试**：外部 API 慢或抖动，直接导致 Agent 工具调用失败。
3. **错误映射**：原始错误信息（堆栈、HTML 错误页）对模型不友好，甚至可能误导。
4. **分页与限流**：很多 API 需要分页，或者有 rate limit，Agent 不会主动处理。
5. **参数校验**：模型可能传入不合法参数，需要在工具侧兜底。

下面用一个简单的例子说明如何把外部订单查询 API 接进 OpenClaw。

## 做法/步骤

### 1. 选择封装方式：MCP 优先

假设我们要对接一个订单查询 REST API：`GET https://api.example.com/v1/orders/{order_id}`，需要 Bearer Token 鉴权。我们写一个轻量的 Python MCP server，暴露 `search_order` 工具。

MCP server 的核心逻辑如下（伪代码，实际可用 FastMCP 或官方 SDK）：

```python
import os
from mcp.server import Server

server = Server("order-api")

@server.tool()
async def search_order(order_id: str) -> dict:
    token = os.environ["ORDER_API_TOKEN"]
    async with httpx.AsyncClient(timeout=10) as client:
        resp = await client.get(
            f"https://api.example.com/v1/orders/{order_id}",
            headers={"Authorization": f"Bearer {token}"}
        )
        if resp.status_code == 404:
            return {"error": "order_not_found"}
        resp.raise_for_status()
        return resp.json()
```

这里做了几件事：从环境变量读 token、显式设置超时、把 404 映射成结构化错误、其他错误抛给上层。

### 2. 在 OpenClaw 中配置 MCP server

在 OpenClaw 的配置文件中加入这一段：

```yaml
mcp:
  servers:
    - name: order-api
      command: python
      args: ["-m", "order_mcp_server"]
      env:
        ORDER_API_TOKEN: ${ORDER_API_TOKEN}
```

这样 OpenClaw 在启动时会拉起这个 MCP server，并把 `search_order` 注册为可用工具。

### 3. 在 Agent 指令中描述工具

在系统提示或工具描述中明确该工具的用途、参数约束和返回格式。例如：

> 查询订单时使用 `search_order` 工具，参数 `order_id` 必须是 8 位数字字符串。如果返回 `order_not_found`，直接告知用户订单不存在，不要重试。

这一步很重要：模型的工具调用质量很大程度取决于描述是否清晰。

### 4. 测试与排错

不要直接接到 Agent 上就完事。先用 MCP Inspector 手动测试工具，确认输入输出符合预期；再在 OpenClaw 中跑几个真实 query，观察工具调用日志。

## 踩坑点

1. **密钥泄漏**：不要在任何提示词、工具描述或返回内容里包含 API 密钥。MCP server 的环境变量是唯一入口。

2. **超时设置缺失**：HTTP 客户端默认超时可能无限等，或者太长。显式设置 5~10 秒，并在超时时返回可理解的错误，让 Agent 能处理。

3. **错误直接透传**：外部 API 返回的 500 页面、nginx 错误页、JSON 里的 traceback 都不应该直接给模型。统一封装成 `{"error": "external_api_unavailable"}` 之类的结构化错误。

4. **分页没处理**：如果 API 返回列表分页，要么在工具内部自动拉取所有页，要么暴露 `cursor`/`page` 参数让 Agent 控制。不要让模型自己去拼 `page=2` 的 URL。

5. **没有重试与熔断**：网络抖动很常见，可以在 MCP server 内增加简单的指数退避重试；对持续失败的 API 做熔断，避免每次调用都卡住 Agent。

6. **参数校验不够**：模型可能传入空字符串、超长字符串、错误类型。在工具入口做一次校验，把非法参数拦截下来，返回明确的参数错误提示。

## 可复用建议

- **统一 API client 层**：不要在每个工具里重复写 HTTP 调用。抽一个 `ApiClient` 类，统一处理鉴权、超时、重试、日志和错误映射。
- **结构化日志**：记录每次外部调用的耗时、状态码、参数（脱敏后）和返回摘要。这样排错时不用猜。
- **先测工具，后接 Agent**：MCP Inspector 能帮你确认工具本身没问题，减少 Agent 侧的干扰变量。
- **敏感字段脱敏**：日志和错误信息里不要出现 token、用户手机号、邮箱等敏感数据。
- **保持幂等**：如果外部 API 支持幂等键，尽量在写操作工具中加上，避免 Agent 重试导致重复创建订单或重复发送通知。

## 总结

Agent 对接外部服务，不是简单地“让模型调一下 API”。它需要一层可靠的工程封装：鉴权隔离、超时控制、错误映射、分页处理、参数校验。MCP 是目前比较顺手的接入方式，能把外部 API 包装成 OpenClaw 的标准工具，同时把复杂性挡在 Agent 之外。

把这层连接层当作基础设施来维护，而不是一次性胶水代码，才能让 Agent 在真实业务里少翻车、可观测、可排查。

---

