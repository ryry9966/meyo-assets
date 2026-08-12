---
title: Agent 与 API 的握手：在 OpenClaw 中用 MCP 封装外部服务的工程实践
feedId: 32730
source: 综合讨论
publishedAt: 2026-08-12
---

## 为什么要让 Agent 能可靠调用外部 API？

Agent 一旦开始读写外部系统（查天气、发消息、取数据库数据），就不再是“封闭的对话大脑”，而是一个脆弱的分布式节点。在本地跑得好好的 Agent，接入一个第三方 API 后就可能出现：超时卡死、token 泄漏、返回格式让模型产生幻觉、限流后自动重试导致雪崩。这些问题本质上不是模型的问题，而是工程链路没有设计好。

OpenClaw 生态里，对接外部服务最稳健的方式不是让 LLM 拼 HTTP 请求，而是通过 **MCP (Model Context Protocol)** 把 API 包装成一组受控的 Tool。这样 Agent 看到的只是一个 function 签名，鉴权、重试、错误转换全部由 MCP Server 处理，Agent 的行为变得可预测。

本文围绕一个真实场景展开：让 OpenClaw Agent 可以查询内部业务数据，API 是已有的 REST 服务（JSON + Bearer Token），对可用性有要求。我们将一步步拆解如何设计、封装、接入并排障。

## 问题拆解：直接调 API 会带来哪些坑？

1. **认证信息泄露与轮换**  
   在 prompt 或 function 描述里硬编码 token，模型可能无意中复述出去，或当上下文变长时被截断后仍留在内存中。更重要的是 token 轮换时，改 prompt 是一种很不可靠的方式。

2. **超时与无响应**  
   Agent 的一次 tool call 如果不设超时，可能会阻塞整个推理循环，用户端看到“正在思考…”直到 HTTP 超时，但进程可能已接近僵死。

3. **返回结构不确定性**  
   大部分 REST API 不会承诺永远返回完全相同的 JSON Schema。增加了一个新字段可能导致 Agent 解析失败，或让模型错误地使用了不可信数据。

4. **限流而非失败**  
   遇到 429 时，如果直接在 tool 内部重试而没有退避，可能会加剧限流，甚至被关闭 key。Agent 也不理解“需要等待”，可能在重试多次后得出错误结论。

5. **缺乏可观测性**  
   每个 tool call 背后的一次 API 调用，如果失败，你在 OpenClaw 的界面只能看到“Tool error”，不知道是超时、鉴权失败还是返回格式错误。

## 做法：用 MCP Server 构建 API 适配层

### 1. 设计 Tool 的契约
以查询业务数据为例，API 是 `GET /api/v1/items?q={keyword}&limit=10`，返回一个 JSON 数组，每个对象有 `id`、`name`、`status`。我们把它封装成一个 MCP tool：`search_items`。

Tool 的输入定义为：
- `keyword` (string, required)
- `limit` (integer, default 10, max 20)

Tool 的输出，我们在 MCP 层做一次“剪裁+校验”：
- 只返回 `id`、`name`、`status`
- 如果原 API 返回了错误字段（如 `error_code`），我们转换成统一的错误结构 `{ error: true, message: "..." }`，不让裸 API 响应直通模型。

### 2. 实现 MCP Server（示例使用 Python FastMCP）

```python
from fastmcp import FastMCP
import httpx
import asyncio
from pydantic import BaseModel, Field

mcp = FastMCP("business-api")

class SearchResponse(BaseModel):
    items: list[dict]
    total: int

async def call_api(keyword: str, limit: int):
    async with httpx.AsyncClient(timeout=10.0) as client:
        resp = await client.get(
            "https://api.internal.example.com/v1/items",
            params={"q": keyword, "limit": limit},
            headers={"Authorization": f"Bearer {API_TOKEN}"}
        )
        resp.raise_for_status()
        data = resp.json()
        # 裁剪字段
        items = [{"id": i["id"], "name": i["name"], "status": i["status"]} for i in data["results"]]
        return SearchResponse(items=items, total=data["count"])

@mcp.tool()
async def search_items(keyword: str, limit: int = 10) -> dict:
    """根据关键词搜索业务条目。limit 最大20。"""
    if limit > 20:
        limit = 20
    try:
        result = await call_api(keyword, limit)
        return result.dict()
    except httpx.HTTPStatusError as e:
        if e.response.status_code == 429:
            return {"error": True, "message": "Too many requests, please wait 30s"}
        return {"error": True, "message": f"Upstream error: {e.response.status_code}"}
    except httpx.TimeoutException:
        return {"error": True, "message": "Request timeout, try again with simpler query"}
```

这里已经解决了几个问题：
- **超时**：`httpx` 设置 10 秒超时，避免无限挂起。
- **限流**：429 不自动重试，直接返回结构化错误让 Agent 知道需要等待。
- **字段裁剪**：防止泄露不要的字段或新增字段干扰模型。
- **错误统一**：任何异常都包装成 `error` 字段，Agent 看到的是可理解的失败，而不是 HTTP traceback。

### 3. 接入 OpenClaw
在 OpenClaw 的 MCP 配置中注册这个 server（假设是通过 stdio 模式）：

```json
{
  "mcpServers": {
    "business-api": {
      "command": "python",
      "args": ["-m", "your_mcp_server"],
      "env": {
        "API_TOKEN": "${API_TOKEN}"
      }
    }
  }
}
```

`API_TOKEN` 通过环境变量注入，不写入任何 prompt 或源文件。权限与加载完全在进程外管理。

Agent 配置里启用 tool `search_items`，然后在 system prompt 中写明用法：“当用户需要搜索条目时，使用 search_items 工具。如果返回 error，告知用户稍候重试。”

## 踩坑点实录

- **tool 描述过于简略导致模型错误传参**  
  初期我们把 `limit` 默认设为 100，但 API 实际只支持最大 50。模型有时会自己编造参数（如 `order_by`）。解决办法是 tool 的 pydantic 描述中明确 boundary，API 端如果有防御性校验，MCP 层也要再次校准。

- **流式输出与 tool call 的时序问题**  
  OpenClaw 在流式生成时，可能在 tool result 回来之前就输出了一部分文本，导致用户看到“正在查询…”然后替换为结果，这本身没问题。但如果 tool 返回 error 却被部分流式文本污染，模型可能忽略 error 继续编造结果。需要在 system prompt 中强调：“只有当工具返回无错误时才引用数据。”

- **重试雪崩**  
  最初我们在 MCP server 内内置了重试（3次），结果发现 Agent 在一次失败的 tool call 后，会自行再次触发同一个 tool call，形成 3×N 的放大。后来改为：只在明确返回瞬时错误（如 503）时才有退避重试一次，其他错误直接返回。让 Agent 的决策层决定是否重试，避免底层暴力重试。

- **没有可观测性**  
  排查为什么 Agent 给出的数据不对时，只看 OpenClaw 的运行日志完全不够。我们在 MCP tool 中加了一行结构化日志（`logger.info("tool_call", extra={"tool":"search_items", "status": e.response.status_code})`），集中到 Loki，很快就能定位是哪一次 API 返回了异常数据。

## 可复用建议

1. **永远加一层中间校验**  
   不要直接把上游 JSON 扔给模型，做一次 pydantic 验证，并抹掉不相关字段。安全且鲁棒。

2. **错误码要“可被 Agent 理解”**  
   返回字符串描述简练、无歧义，例如 `"Rate limited. Retry after 30 seconds"` 而非 `"429 Too Many Requests"`。

3. **配置外部化**  
   API 地址、超时、token、重试策略都放在 MCP server 的环境变量或配置文件里，不要写死在代码或 Agent 描述中。

4. **对幂等性敏感的操作（写操作）额外谨慎**  
   例如 `POST` 提交订单，一定要把 tool 设计成“预提交”+“确认”两段式，避免模型在重试时产生重复订单。

5. **在 OpenClaw 端做 tool use 的 budget 控制**  
   如果一次对话可能多次调用同一 API，可以设置 max steps 限制，避免死循环。

## 总结

让 Agent 与外部服务握手，关键不在于让模型学会发 HTTP 请求，而在于为它构建一条 **坚固、可观测的边界**。MCP Tool 正是这个边界的最佳载体：它把不可控的 API 世界转化为一组强类型的函数，同时隔离鉴权、超时和错误扩散。在 OpenClaw 里，花一点时间打磨这个适配层，后续任何 Agent 行为的不确定性都会大幅收敛。

把 MCP 当成一个“反脆弱”的网关来设计，你的 Agent 就能从一只美丽的玻璃动物，变成真正能干活的后端服务员。

---

