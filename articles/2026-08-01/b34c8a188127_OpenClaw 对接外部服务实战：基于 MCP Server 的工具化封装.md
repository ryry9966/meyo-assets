---
title: OpenClaw 对接外部服务实战：基于 MCP Server 的工具化封装
feedId: 31198
source: 综合讨论
publishedAt: 2026-08-01
---

## 背景

OpenClaw 作为面向智能体的运行时框架，核心能力之一就是让 Agent 能安全地调用外部服务。无论是查询数据库、发送 HTTP 请求还是控制 IoT 设备，Agent 都不能直接“裸调”，必须有清晰的边界：鉴权、参数校验、错误映射、超时与重试。MCP (Model Context Protocol) 的出现恰好为这类“Agent ↔ 工具”的交互提供了一套标准化方案。

在工程实践中，我们面临一个具体问题：如何让 OpenClaw 的 Agent 快速接入一个已有的第三方 REST API（例如内部 CMDB），同时保证调用可观测、可复现，且不把密钥暴露在 prompt 里。下面我会基于 OpenClaw 的 MCP 支持，拆解从 0 到 1 的对接步骤，并分享踩过的坑和可复用的封装思路。

## 问题拆解

对接外部服务时，常见痛点有：
- 工具定义为 Python 函数容易与业务耦合，难以跨 Agent 复用。
- 密钥、endpoint 等敏感信息容易泄漏到对话上下文或日志。
- 网络抖动、上游异常时缺少统一的重试和降级策略。
- 调用链路缺乏 trace，排障时需要翻多个系统的日志。

MCP 本质上定义了 Agent 与工具服务器之间的标准化通信，OpenClaw 内置 MCP 客户端，可以直接连接符合 MCP 协议的 server。我们把外部 API 封装为一个独立的 MCP server，OpenClaw 负责发现工具并调用，这样问题就被拆成了两个清晰的工程任务：编写 MCP server，以及在 OpenClaw 中配置连接。

## 实践步骤

以下步骤均假设你已有一个 OpenClaw 可用环境（版本 `>=0.4`），且使用 Python 生态。

### 1. 编写 MCP Server（以 Python 为例）

目标：封装一个查询 CMDB 主机信息的 REST API。新建项目 `mcp-cmdb`，使用 `mcp` 库快速构建：

```python
# server.py
import os, httpx
from mcp.server import Server, NotificationOptions
from mcp.server.models import InitializationCapabilities
from mcp.server.stdio import stdio_server
import mcp.types as types

server = Server("cmdb")

@server.list_tools()
async def list_tools() -> list[types.Tool]:
    return [
        types.Tool(
            name="query_host",
            description="根据主机名或 IP 查询 CMDB 中的主机信息",
            inputSchema={
                "type": "object",
                "properties": {
                    "query": {"type": "string", "description": "主机名或 IP"}
                },
                "required": ["query"]
            }
        )
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[types.TextContent]:
    if name != "query_host":
        raise ValueError(f"Unknown tool: {name}")
    query = arguments["query"]
    base_url = os.environ["CMDB_BASE_URL"]
    token = os.environ["CMDB_API_TOKEN"]
    async with httpx.AsyncClient(timeout=10) as client:
        resp = await client.get(
            f"{base_url}/api/v1/hosts",
            params={"search": query},
            headers={"Authorization": f"Bearer {token}"}
        )
        resp.raise_for_status()
        data = resp.json()
        # 仅返回必要字段，避免返回大段无用信息
        content = json.dumps(data, ensure_ascii=False)
    return [types.TextContent(type="text", text=content)]

async def main():
    async with stdio_server() as (read_stream, write_stream):
        await server.run(
            read_stream,
            write_stream,
            InitializationCapabilities(
                sampling={},
                experimental={},
                roots={"listChanged": True}
            ),
            notification_options=NotificationOptions(),
        )

if __name__ == "__main__":
    import asyncio, json
    asyncio.run(main())
```

要点：
- 所有敏感信息通过环境变量注入，绝不硬编码。
- `inputSchema` 严格声明参数，方便 OpenClaw 自动生成 function call 的 schema。
- 工具内部做好 `timeout`，避免上游服务拖死整个 Agent 调用链。

### 2. 在 OpenClaw 中注册 MCP Server

OpenClaw 的 Agent 配置通常为 YAML/JSON 文件，添加一个 MCP client 节点：

```yaml
agent:
  name: ops-assistant
  mcp_servers:
    - name: cmdb
      command: python
      args: ["/path/to/mcp-cmdb/server.py"]
      env:
        CMDB_BASE_URL: "https://cmdb.internal.example.com"
        CMDB_API_TOKEN: "${CMDB_TOKEN}"   # 从 secret manager 注入
```

Agent 启动时会自动启动子进程运行 MCP server，并通过 stdio 建立双向 JSON-RPC 通道。随后在对话中，Agent 就可以直接调用 `query_host` 工具，OpenClaw 的 planner 会根据工具描述决定何时调用。

### 3. 验证与调试

可以在 OpenClaw 的 CLI 中直接测试：
```
openclaw run ops-assistant "帮我查一下主机 prod-db-01 的配置"
```
期望 Agent 返回 JSON 信息，并以自然语言总结。

如果调用失败，先检查 MCP server 的日志（OpenClaw 会捕获子进程 stderr 并输出到自己的日志中）。常见错误包括：环境变量未设置、上游 API 返回 4xx/5xx、网络不可达等。

## 踩坑记录

1. **子进程生命周期管理**  
   早期版本里，如果 MCP server 意外退出，OpenClaw 不会自动重启，导致后续工具调用全部失败。解决方案是在 OpenClaw 的 server manager 中开启 `auto_restart: true`（需版本 >=0.4.1），或使用外部进程守护（如 systemd）。

2. **参数序列化问题**  
   `call_tool` 接收到的 `arguments` 已经是 Python dict，但如果你的 schema 里允许嵌套对象，注意 OpenClaw 传来的可能是 JSON 字符串，需用 `json.loads` 处理一次。统一在内层做 `if isinstance(arguments, str): arguments = json.loads(arguments)` 防御即可。

3. **tool 描述不足导致误调用**  
   Agent 是否正确地调用工具，严重依赖 `Tool.description` 和 `inputSchema.description` 的质量。描述要具体，避免“查询主机”。应该写成：“根据主机名（如 prod-db-01）或 IP 地址查询 CMDB，返回硬件配置、所属环境、负责人等信息”，这样 Agent 才知道何时使用。

4. **错误信息泄漏**  
   如果 `call_tool` 抛出未捕获的异常，MCP 会返回原始 traceback，容易被 Agent 追问出敏感内容。务必在最外层 `try...except`，返回结构化的错误信息，例如：
   ```python
   return [types.TextContent(type="text", text=json.dumps({"error": "upstream unavailable"}))]
   ```

## 可复用建议

- **标准化 Tool Schema 模板**：团队内部可以抽象一个 `BaseTool` 类，要求每个工具程序员必须遵守 `inputSchema` 的 JSON Schema 规范，并统一错误结构为 `{"error": "message", "code": "UNKNOWN"}`。
- **统一配置注入**：敏感信息通过 OpenClaw 的 `env` 字段注入，但其值应来自外部 secret manager（如 Vault），不要在配置文件明文书写。OpenClaw 支持 `${VAR}` 语法，可直接引用系统环境变量。
- **加入可观测性**：在 MCP server 内部对每个 `call_tool` 入口添加结构化日志（包含 tool_name, arguments, latency, status），便于后续整合到 tracing 系统。
- **离线测试**：编写 MCP server 的单元测试时，可以直接用 `mcp.client` 发起模拟请求，不需要启动整个 OpenClaw。这样可以快速验证工具逻辑。

## 总结

OpenClaw 通过 MCP 将外部服务接入 Agent 的做法，本质是把不可控的 API 调用变成可控的工具声明。工程收益明显：Agent 不需要知道 HTTP、Token 这些东西，只需要理解工具的“语义契约”；运维侧则获得了统一的权限控制、错误处理和监控埋点。

在真实项目中，对接三个以上的外部服务时，这种“一个服务一个 MCP server”的模式会带来一些进程管理成本。可以考虑用轻量的网关模式，在一个 MCP server 内部根据路由分发到不同后端，但要注意别让它变成新的单体。无论如何，先将工具接入机制标准化，是让 Agent 落地的第一步，也是最关键的一步。

---

