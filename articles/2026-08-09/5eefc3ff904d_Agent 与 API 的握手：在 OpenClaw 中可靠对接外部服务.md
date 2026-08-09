---
title: Agent 与 API 的握手：在 OpenClaw 中可靠对接外部服务
feedId: 32207
source: 综合讨论
publishedAt: 2026-08-09
---

## 背景

Agent 的能力边界很大程度上由它能调用的外部服务决定。无论是查询数据库、触发 CI/CD、读取实时天气，还是推送消息到企业微信，都依赖 Agent 与 API 之间的“握手”。在 OpenClaw 的体系里，这个握手主要通过 **MCP（Model Context Protocol）工具** 完成。MCP 屏蔽了底层 transport，让 Agent 把外部 API 当成一组标准的 `tool` 来消费。

但工程落地时，问题并不出在协议本身，而出在握手前后的细节：认证怎么管理、超时/重试怎么设计、返回的 JSON 如何让大模型稳定理解。本文以“对接一个企业内部审批状态查询 API”为例，拆解完整的接入路径，并给出可复用的实践。

## 问题场景

我们有一个内部审批系统，提供 RESTful API：`GET /approval/{id}`，返回 `{"status":"approved|pending|rejected","approver":"...","update_time":"..."}`。期望 OpenClaw Agent 能按需查审批状态，并据此决定是否继续流水线。

直观的做法是写一个 Python 脚本封装 API，然后在 Agent 的 prompt 里手写 function call 描述，让模型生成调用指令再执行。这样做有两个痛点：

1. **认证泄露风险**：Bearer Token 容易随 prompt 或日志暴露。
2. **错误处理脆弱**：网络抖动或返回非预期结构时，Agent 容易“失明”，进入幻觉循环。

用 MCP 工具的方式来解决，可以把 API 细节彻底封装在 server 侧，Agent 只看到干净的函数签名。

## 做法与步骤

### 1. 定义 MCP Server

在 OpenClaw 中，MCP server 可以是任意语言实现的进程，通过 stdio 或 HTTP（server-sent events）与客户端通信。这里以 Python 为例，使用 `mcp` SDK：

```python
# approval_server.py
import os
import httpx
from mcp.server import Server
from mcp.types import Tool, TextContent

server = Server("approval")

@server.list_tools()
async def list_tools():
    return [
        Tool(
            name="get_approval_status",
            description="查询指定审批单的状态。",
            inputSchema={
                "type": "object",
                "properties": {
                    "approval_id": {"type": "string"}
                },
                "required": ["approval_id"]
            }
        )
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict):
    if name == "get_approval_status":
        approval_id = arguments["approval_id"]
        token = os.getenv("APPROVAL_API_TOKEN")
        if not token:
            return [TextContent(type="text", text="Error: missing API token")]
        async with httpx.AsyncClient(timeout=8) as client:
            resp = await client.get(
                f"https://api.internal/approval/{approval_id}",
                headers={"Authorization": f"Bearer {token}"}
            )
        if resp.status_code != 200:
            return [TextContent(
                type="text",
                text=f"API error {resp.status_code}: {resp.text[:200]}"
            )]
        data = resp.json()
        # 返回结构化的、大模型友好的文本
        return [TextContent(
            type="text",
            text=f"审批单 {approval_id} 状态：{data['status']}，"
                 f"审批人：{data['approver']}，更新时间：{data['update_time']}"
        )]
```

关键点：认证 Token 从环境变量读取，绝不硬编码；返回给模型的是自然语言摘要，而非原始 JSON，减少解析歧义。

### 2. 在 OpenClaw 中注册 MCP Server

在 OpenClaw 的 agent 配置里，添加一个 MCP 连接：

```yaml
agents:
  pipeline_agent:
    mcp_servers:
      - name: approval
        command: python
        args: ["approval_server.py"]
        env:
          APPROVAL_API_TOKEN: ${APPROVAL_API_TOKEN}
```

OpenClaw 启动 Agent 时会自动拉起这个 MCP server 进程，并完成工具发现。对 Agent 而言，它看到的就是一个叫 `get_approval_status` 的函数，可以直接在对话中调用。

### 3. Agent 里的调用链路

在 OpenClaw 的 agent 定义中，不需要额外写 prompt 来描述工具——MCP 已经将 `Tool` 的描述和 schema 注入给模型。你只需要告诉 Agent 什么时候该用：

```
你是一个流程管家。当用户询问某个审批单状态时，使用 get_approval_status 工具查询，并根据返回的状态执行下一步动作。
```

当用户说“帮我查 PRD-2024-0991 的状态”，Agent 自动生成一次 tool call，MCP 客户端将请求路由到 `approval_server.py`，执行 HTTP 请求，结果返回给模型，模型再将结果以自然语言呈现。

## 踩坑与排障

1. **Token 过期未刷新**  
   MCP server 是长生命周期的进程，若 API Token 在 server 运行期间过期，所有调用都会崩。解决方式：在 server 内部加一个 token 刷新逻辑，或配合外部定时重启；切忌在每次 tool call 里实时获取短期 token（开销大）。更稳健的做法是让服务器返回 401 时，通过 `return` 明确告知 Agent “认证已失效，请通知管理员”，而非静默失败。

2. **超时与重试**  
   在 `httpx` 层设置超时（如上例 8 秒）是基础。但注意 OpenClaw 的 MCP 客户端自身也有调用超时（默认 60 秒）。如果 API 偶尔慢响应，在 server 内做一次重试（带退避）会更友好。不要在 server 中无限重试，否则会阻塞 Agent 整体思考流。

3. **错误传播的语义**  
   当 API 返回 404 或 500 时，不要直接抛异常导致 MCP server 退出。应该在 `call_tool` 中捕获，并返回明确的错误文本，让 Agent 有机会理解并给出补救建议（如“审批单不存在，请检查单号”）。模型看到结构化的错误信息，比看到 `"An error occurred"` 更能做出正确行动。

4. **大模型对长串数字/ID 的幻觉**  
   一些 API 返回的数据中包含长 ID。尽量在返回文本中保留前几位和后几位，中间打码；或直接返回人类可读摘要，避免模型“背下”完整 ID 后在后续回复中篡改。

## 可复用建议

- **优先使用 MCP Registry**：OpenClaw 支持从社区 registry 安装 MCP server，如果有现成的，直接用，减少自研成本。
- **工具描述即文档**：在 `Tool.description` 里写清楚成功/失败返回格式，帮助模型建立预期。对复杂 API，可在 description 中提供典型输入输出样例。
- **结构化输出兜底**：如果后续想让 Agent 根据状态触发流程跳转，可让 MCP 工具同时返回一个隐含的结构化标记（如 `[STATUS:approved]`），在 agent 指令里用少量规则解析，比完全依赖模型判断更可靠。
- **安全边界**：为 MCP server 配置最小权限，API Token 作用域仅限必需接口；通过 OpenClaw 的环境变量注入机制隔离敏感信息，不要写进配置文件仓库。

## 总结

Agent 与外部 API 的握手，本质是**将不可靠的外部资源封装成可靠的内部原语**。OpenClaw 的 MCP 机制让这一封装变得标准化：API 的认证、重试、格式转换全部发生在 Server 进程内，Agent 只看到刚好的信息。这套模式复制到企业通知、审批、监控、数据查询等场景几乎零修改——只要你会写一个单文件的 Python/Node.js 脚本，就能为 Agent 赋予新的手脚。

下次你的 Agent 需要“伸手”碰外部系统时，不妨先建一个 MCP 工具，写稳那层握手逻辑。剩下的事，让 OpenClaw 去调度。

---

