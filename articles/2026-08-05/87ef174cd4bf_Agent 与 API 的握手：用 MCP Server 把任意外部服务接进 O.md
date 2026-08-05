---
title: Agent 与 API 的握手：用 MCP Server 把任意外部服务接进 OpenClaw
feedId: 31759
source: 综合讨论
publishedAt: 2026-08-05
---

## 背景：Agent 的“最后一公里”总是卡在外部服务上

不管是做自动化工作流、数据处理流水线，还是一个能查订单状态的内部机器人，Agent 的瓶颈往往不是推理能力，而是如何稳定、可控地访问外部服务。

OpenClaw 允许你通过 Tool 或插件扩展 Agent 能力，但真正落到工程里，大部分场景都是把已有的 REST API、gRPC 接口甚至数据库查询包装成 Agent 可调用的工具。如果每次都把认证头、超时、重试、错误处理直接写在 Agent 的调用逻辑里，很快就会变成一坨难以维护的胶水代码。

最近社区里比较务实的做法是：把外部服务抽象成一个独立的 MCP (Model Context Protocol) server，然后让 OpenClaw 通过 MCP 协议去调用这个 server。这样 API 的握手逻辑完全沉淀在 server 层，Agent 只关心工具声明和业务参数。

## 问题拆解

一个典型的对接场景大概长这样：

- 上游：公司内部有一个“工单查询 API”，需要 Bearer Token 鉴权，返回 JSON。
- 下游：OpenClaw 上跑着一个工单助手 Agent，需要根据用户输入查询工单状态。
- 目标：把这个 API 变成 Agent 可用的工具，但不把 token 写进 prompt 里，也不让 Agent 直接处理 401/限流等 HTTP 细节。

我们要做的就是在中间加一层 MCP server，做一个稳定的握手层。

## 实操步骤：从零到一接一个外部 API

### 1. 起一个最小 MCP server

用 Python 和 `mcp` 官方库快速搭建。假设我们要暴露一个 `query_ticket` 工具：

```python
import asyncio
import os
import httpx
from mcp.server import Server, NotificationOptions
from mcp.server.models import InitializationCapabilities
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent

server = Server("ticket-mcp")
API_BASE = os.environ.get("TICKET_API_BASE", "https://api.internal.example.com")
API_TOKEN = os.environ.get("TICKET_API_TOKEN", "")

@server.list_tools()
async def list_tools() -> list[Tool]:
    return [
        Tool(
            name="query_ticket",
            description="按工单ID或用户ID查询工单状态",
            inputSchema={
                "type": "object",
                "properties": {
                    "ticket_id": {"type": "string", "description": "工单ID"},
                    "user_id": {"type": "string", "description": "用户ID"}
                },
                "required": []
            }
        )
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    if name == "query_ticket":
        headers = {"Authorization": f"Bearer {API_TOKEN}"}
        params = {k: v for k, v in arguments.items() if v is not None}
        # 核心外部调用，带超时和浅层重试
        async with httpx.AsyncClient(timeout=15) as client:
            for attempt in range(3):
                try:
                    resp = await client.get(
                        f"{API_BASE}/tickets",
                        params=params,
                        headers=headers
                    )
                    resp.raise_for_status()
                    data = resp.json()
                    return [TextContent(type="text", text=json.dumps(data, ensure_ascii=False))]
                except httpx.TimeoutException:
                    if attempt == 2:
                        return [TextContent(type="text", text="查询超时，请稍后重试")]
                    await asyncio.sleep(2 ** attempt)
                except httpx.HTTPStatusError as e:
                    if e.response.status_code == 401:
                        return [TextContent(type="text", text="服务认证失败，请联系管理员")]
                    if e.response.status_code == 429:
                        if attempt == 2:
                            return [TextContent(type="text", text="请求过于频繁，请稍后重试")]
                        await asyncio.sleep(5)
                    else:
                        return [TextContent(type="text", text=f"查询失败：{e.response.status_code}")]
                except Exception as e:
                    return [TextContent(type="text", text=f"未知错误：{str(e)}")]
        return [TextContent(type="text", text="未知错误")]
    raise ValueError(f"Unknown tool: {name}")

async def main():
    async with stdio_server() as (read_stream, write_stream):
        await server.run(
            read_stream,
            write_stream,
            InitializationCapabilities(
                sampling={},
                experimental={},
            ),
            notification_options=NotificationOptions(),
        )

if __name__ == "__main__":
    asyncio.run(main())
```

注意几个细节：

- 敏感信息（token、base URL）全部走环境变量，不硬编码。
- `call_tool` 里做了三层防护：超时兜底、429 限流指数退避、401 直接告知失败而不泄漏 token。
- 工具返回的是纯文本 JSON，Agent 可以直接理解，避免又嵌套一层解析。

### 2. 在 OpenClaw 里接入这个 MCP server

OpenClaw 的 MCP 集成通常通过 `mcp.json` 或者界面配置。假定用配置文件：

```json
{
  "mcpServers": {
    "ticket-api": {
      "command": "python",
      "args": ["path/to/ticket_server.py"],
      "env": {
        "TICKET_API_BASE": "https://api.internal.example.com",
        "TICKET_API_TOKEN": "${TICKET_API_TOKEN}"
      }
    }
  }
}
```

这里 `${TICKET_API_TOKEN}` 建议从宿主机环境变量或密钥管理器注入，别直接写进配置仓库。

重启 OpenClaw 后，在 Agent 设置页面就能看到新工具 `query_ticket`，可以手动测试一下调用是否正常。

### 3. 让 Agent 调用

Agent 配置里指定可用的工具白名单，只暴露它需要的那个 MCP tool。然后跑一个简单测试：问“帮我查下工单 T-1234 的状态”，预期 Agent 会识别到需要调用 `query_ticket` 并传入 `ticket_id = "T-1234"`。

## 踩坑点与排障

**坑1：MCP stdio 进程管理**  
最常见的问题是 MCP server 子进程僵死或 stdout 被污染。尽量不要在 server 里用 `print()` 打调试日志，改用 `logging` 输出到 stderr，或者在 `call_tool` 里显式返回错误信息。否则 OpenClaw 的 MCP 客户端可能因为不合规的 JSON-RPC 消息而断连。

**坑2：超时链路过长**  
Agent 调用 OpenClaw 有一个超时，OpenClaw 调用 MCP 又有一个超时，MCP 内部还有 HTTP 超时。三层超时如果不协调，大概率会出现 Agent 端报错“工具调用超时”，但实际 server 还在跑。建议 MCP server 内部超时设为略小于 OpenClaw 侧工具调用超时的值，同时服务器自身设置一个总超时（比如 20s）。

**坑3：认证失败的信息泄露**  
不要在给 Agent 的返回里包含原始 HTTP 响应体或 stack trace，否则容易把 token 或内部域名泄露到对话记录里。上面例子里直接返回通用错误信息且打了日志在 server 端，是一个安全实践。

**坑4：并发与幂等**  
如果这个 API 是写操作（比如创建工单），请确保 MCP 调用具备幂等性，并且在 OpenClaw 端不要开启过激的自动重试。外部服务的写入重试要由 server 侧的 token 或唯一请求 ID 控制，否则可能出现重复创建。

## 可复用建议

1. **抽象出统一的 MCP tool 模板**：大多数 REST API 都可以套“query_xxx”“create_xxx”这样的工具模式。写一个基类或模板函数，把鉴权、超时、重试、结构化返回都封装好，然后具体工具只需补充 schema 和业务逻辑。

2. **环境变量管理**：用 `.env` + `direnv` 或容器注入，严禁将 credential 提交到 Git。OpenClaw 的 server 配置里可以引用宿主环境变量，注意在 CI/CD 中也要注入对应 secret。

3. **工具描述写精确**：OpenClaw 依赖工具描述来决定何时调用，所以 `Tool.description` 和参数的 `description` 必须写清楚字段含义、格式和限制。模糊的描述会导致 Agent 乱传参数或者不调用。

4. **可观测性兜底**：在 MCP server 里加基本的调用量、失败率统计（打印到 stderr 或推送到监控），当 Agent 说“我查不到”时，你能快速判断是 API 挂了还是 Agent 没调用。

## 总结

对接外部服务不是简单的 import + fetch，而是要把“易变的外部接口”和“要求稳定的 Agent 行为”隔离开。MCP server 层相当于一个握手协议转换器，让 API 的错误、限流、鉴权都在可控范围内消化。

工程上只需要三个动作：

- 用 `mcp` 库起一个 stdio server，做好超时和错误处理。
- 在 OpenClaw 里配置 mcpServers，注入密钥。
- Agent 工具界面勾选目标工具，prompt 里一句话引导。

之后你就可以把任意内部 API 变成 Agent 的稳定手掌，这套模式在数据库查询、文档检索、操作 OSS 等场景下同样适用。别再让 Agent 直接裸调 API 了，给它一个稳定的 MCP 中间层吧。

---

