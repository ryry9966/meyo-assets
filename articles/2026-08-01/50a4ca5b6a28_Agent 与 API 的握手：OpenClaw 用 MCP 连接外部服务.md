---
title: Agent 与 API 的握手：OpenClaw 用 MCP 连接外部服务
feedId: 31246
source: 综合讨论
publishedAt: 2026-08-01
---

## 背景

Agent 开始从“会用工具”走向“能干活”，但真正落地的第一步，往往卡在一件小事上：它怎么安全、可靠地调用一个已有的 HTTP API？

直接让模型构造 curl 或拼 JSON，不稳定；把 API key 写进 system prompt，不安全；让 Agent 不断重试直到成功，不可控。OpenClaw 给出的答案是 MCP（Model Context Protocol）——把每一组外部服务封装成标准化的工具服务器，Agent 只负责调用，不碰协议细节。

这篇文章围绕一个真实需求（查询企业内部订单状态），展示如何用 FastMCP 把外部 REST API 包装成 OpenClaw 可用的工具，并整理出工程上容易踩的坑与可复用的建议。

## 问题拆解

我们希望 Agent 能回答：“帮我查一下订单 ORD-20240321-001 的状态”。后台有一个现成的内部 Order API，接收 GET /orders/{id}，返回 JSON，需要 Bearer Token 鉴权。

直接让大模型发起 HTTP 请求的风险包括：
- 鉴权信息泄露或被模型误用；
- 模型可能编造参数、路径、方法；
- 请求失败后没有统一重试/降级策略；
- 响应体过大可能撑爆上下文窗口。

所以，正确的做法是：**把 API 变成有明确输入、输出的工具函数，放在 Agent 外运行，受工程约束。**

## 做法与步骤

### 1. 用 FastMCP 封装 API 为 MCP Tool

创建一个 Python MCP 服务器，使用 `fastmcp` 库（假设 OpenClaw 生态内已有）。我们关心三个细节：鉴权隔离、响应裁剪、错误重试。

```python
# mcp_server_orders.py
import os, httpx
from fastmcp import FastMCP

mcp = FastMCP("OrderService")

ORDER_API_BASE = os.getenv("ORDER_API_BASE", "https://api.internal/orders")
ORDER_API_TOKEN = os.getenv("ORDER_API_TOKEN")  # 仅在这里注入，不暴露给模型

async def fetch_order(order_id: str) -> dict:
    headers = {"Authorization": f"Bearer {ORDER_API_TOKEN}"}
    async with httpx.AsyncClient(timeout=10) as client:
        for attempt in range(3):
            try:
                resp = await client.get(f"{ORDER_API_BASE}/{order_id}", headers=headers)
                resp.raise_for_status()
                data = resp.json()
                # 裁剪：只保留 Agent 需要的字段，避免上下文溢出
                return {
                    "order_id": data.get("id"),
                    "status": data.get("status"),
                    "updated_at": data.get("updated_at"),
                    "items_count": len(data.get("items", []))
                }
            except Exception as e:
                if attempt == 2:
                    raise
                await asyncio.sleep(1)

@mcp.tool()
async def get_order_status(order_id: str) -> str:
    """查询订单最新状态。输入订单ID（如 ORD-20240321-001），返回 JSON 字符串。"""
    try:
        order = await fetch_order(order_id)
        return json.dumps(order, ensure_ascii=False)
    except Exception as e:
        return json.dumps({"error": f"查询失败：{str(e)}"})

if __name__ == "__main__":
    mcp.run()
```

要点：
- API Token 通过环境变量注入，工具描述里绝不出现密钥。
- 函数签名用类型标注，FastMCP 会自动生成 input schema，约束 Agent 传参。
- 三次重试，每次间隔 1 秒，最终失败仍返回可读的错误 JSON，避免 Agent 陷入死循环。
- 返回值裁剪到仅 4 个关键字段，减少 token 消耗。

### 2. 在 OpenClaw 中注册该 MCP 服务

编辑 `mcp_servers.json` 或在 OpenClaw 控制台添加：

```json
{
  "mcpServers": {
    "order_service": {
      "command": "python",
      "args": ["/path/to/mcp_server_orders.py"],
      "env": {
        "ORDER_API_BASE": "https://api.internal/orders",
        "ORDER_API_TOKEN": "${ORDER_API_TOKEN}"
      }
    }
  }
}
```

启动 OpenClaw 时，它会自动拉起这个 Python 进程，通过标准 MCP 协议通信。

### 3. 配置 Agent 使用工具

在 Agent 定义中，将 `order_service` 加入工具列表，并在 system prompt 中添加简单指引：

> 你可以使用 get_order_status 工具查询订单状态，参数为订单编号。

之后 Agent 就可以通过函数调用来查询订单了。整个过程中，模型完全没有接触到 token，也无法直接访问内部域名。

## 踩坑记录

**坑1：工具返回值被模型二次加工丢失精度**

即使返回了 JSON 字符串，模型有时会“好心”总结，导致丢失字段。解决办法是在 prompt 明确要求：“当调用工具得到结果后，请原样输出订单状态和更新时间，不要总结。”也可以在工具描述结尾加一句：“返回结果应直接展示给用户。”

**坑2：MCP 进程僵死**

长时间没有工具调用，Python 进程可能被容器或操作系统杀死。可以通过在 FastMCP 内部添加心跳日志，或者配置 OpenClaw 的健康检查。另一种办法是用 systemd 或 supervisor 管理进程，并在 mcp_servers 里改为调用一个守护脚本。

**坑3：高并发时多个 Agent 共用同一个 MCP 服务**

一个 Order API 服务器可能需要承载多个会话发来的请求。FastMCP 默认同步处理每个工具调用，如果 API 响应慢，会阻塞后续调用。需要确保工具函数是 async 的，且 httpx 使用 AsyncClient 连接池，避免雪崩。

**坑4：错误信息过于冗长**

直接返回 `repr(e)` 会把堆栈信息全部丢给模型，浪费 token 且可能泄露敏感路径。应该用自定义的错误摘要，如上面代码所示。

## 可复用建议

1. **统一工具架子**：写一个 BaseTool 类，内置日志、重试、超时、截断逻辑，所有 API 封装继承它，只写核心 fetch 逻辑。
2. **测试先行**：在 OpenClaw 的工具测试页面，先用固定参数跑几次，确认 schema 和返回格式符合预期，再让 Agent 调用。
3. **区分读写**：对写操作（取消订单、更新信息）单独封装，并在 Agent 侧要求用户二次确认。可以在工具层面实现一个简单的审批流，例如写操作返回 `{"action": "confirm_required", "preview": {...}}`，由 Agent 引导用户确认。
4. **观察性**：将每次工具调用的参数、耗时、返回状态记录到日志，方便排查 Agent 错误决策或者外部 API 抖动。
5. **缓存静态数据**：如果某个 API 的数据变化不频繁，可以考虑在 MCP 服务里加一层短时缓存，既能降低外部负载，也能让 Agent 获得更快的响应。

## 总结

Agent 与外部服务的握手，本质上就是一次工程边界的定义：谁管鉴权、谁管重试、谁管上下文大小。OpenClaw 的 MCP 架构把这些问题关在了工具服务器里，让 Agent 面对的是声明式、可测试的函数，而不是脆弱的 HTTP 细节。

当你开始把一个又一个 API 封装成 MCP 工具时，会发现 Agent 的可靠度会显著提升——不是因为模型变聪明了，而是因为不确定的外部世界被工程手段约束住了。

---

