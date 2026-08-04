---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 31648
source: 综合讨论
publishedAt: 2026-08-05
---

## 背景：Agent 时代，工具集成成了新的 Babel 塔

过去一年做 OpenClaw 插件和自动化流程，我相信你和我遇到的是同一个问题：每个 Agent 框架都在发明自己的工具调用格式。OpenAI 有 function calling，Claude 有 tool use，我们自己写的框架又要一套 JSON Schema 描述。每个模型服务商提供的工具描述、参数校验、结果回传格式都不完全一致。

这意味着，如果你想让 Agent 同时调用 GitHub API、查询本地数据库、操作浏览器，你实际上要写三套不同的适配层。每接一个新的模型或新的工具，都要重新做一遍胶水代码。

MCP（Model Context Protocol）要解决的就是这个问题：把"模型怎么描述工具"和"工具怎么暴露能力"这两件事标准化。它不是新的插件格式，不是 SDK，而是一个协议层。

## 问题：工具调用的上下文断裂

在本地跑 OpenClaw 的 agent 时，最麻烦的点不是模型能力不够，而是**上下文断裂**。

举个例子：Agent 需要查一个用户订单，然后根据订单内容发送一封邮件。传统做法是：

1. 你告诉 Agent 有一个 `query_order(order_id: string)` 函数
2. Agent 决定调用它，模型返回一个 JSON 字符串
3. 你的主程序解析这个 JSON，执行函数，拿到结果
4. 把结果塞回对话历史，让模型继续

听起来不复杂，但一旦工具数量超过 10 个，问题就爆发了：

- **描述膨胀**：每个工具需要详细的描述、参数说明、枚举值。模型每次请求都要把这些信息重新读一遍，token 开销大，响应变慢
- **上下文污染**：工具返回的原始数据未经结构化处理，直接拼进对话历史，模型容易被不相关字段带偏
- **权限混乱**：代码里到处是 `if tool_name == "send_email":` 这种分支判断，每加一个新工具就要改主逻辑

MCP 的做法是把这些结构性问题抽象成三个角色：

- **MCP Host**：运行 Agent 的进程（比如 OpenClaw）
- **MCP Client**：Host 与工具服务之间的连接器
- **MCP Server**：一个独立的进程或服务，暴露工具，维护上下文

Server 自己管理工具的定义和生命周期，Host 通过标准协议发现问题、调用工具、处理结果。Agent 不需要知道工具是用 Python 写的还是 Node 写的，只需要知道 MCP 协议。

## 做法：30 分钟跑通一个最小 MCP Server

以 OpenClaw 为例，你的 agent 本身不需要直接实现 MCP Server，只需要让 Host 连接到已有的 MCP Server 即可。

我这里用一个最简的 Python 示例，不需要任何重框架：

```python
# mcp_server.py
# 使用官方 mcp 库（pip install mcp）

from mcp.server import Server
from mcp.server.stdio import stdio_server
import anyio

app = Server("demo-server")

@app.list_tools()
async def list_tools():
    return [
        {
            "name": "get_local_time",
            "description": "获取指定时区的当前时间",
            "inputSchema": {
                "type": "object",
                "properties": {
                    "timezone": {"type": "string"}
                }
            }
        }
    ]

@app.call_tool()
async def call_tool(name: str, arguments: dict):
    if name == "get_local_time":
        from datetime import datetime
        import zoneinfo
        tz = zoneinfo.ZoneInfo(arguments.get("timezone", "Asia/Shanghai"))
        return {"time": datetime.now(tz).isoformat()}

async def main():
    async with stdio_server() as (read_stream, write_stream):
        await app.run(read_stream, write_stream, app.create_initialization_options())

if __name__ == "__main__":
    anyio.run(main)
```

然后在 OpenClaw 的配置里声明这个 server：

```yaml
# openclaw.yaml
mcp:
  servers:
    demo:
      command: "python"
      args: ["mcp_server.py"]
```

启动 OpenClaw，Agent 就能自动发现 `get_local_time` 这个工具，不需要你在 agent 里写任何胶水代码。

## 踩坑点

**1. 工具返回结果要结构化，别返回纯文本**

我一开始图省事，工具直接返回 `"done"` 或者一段错误文本。MCP 协议本身允许返回文本，但 Agent 会把它当成自由文本进行"理解"，有时会编造不存在的结果。正确做法是返回结构化数据：

```json
{"ok": true, "data": {"order_id": "123", "status": "shipped"}}
```

**2. 长上下文问题是协议解决不了的**

MCP 标准化了工具调用格式，但它不负责 token 管理。如果某个工具返回一个 10 万元的表格，Agent 照样爆炸。你需要自己实现一次结果摘要或字段裁剪层。

**3. 注意权限边界**

MCP Server 是独立进程，意味着它拥有与你的主进程同等的权限。如果一个 MCP Server 是从 pip/npm 安装的第三方包，它可以直接访问你文件系统。别给 MCP Server 配置过大的 API key 权限，像对待外部依赖一样对待它。

**4. stdio vs SSE vs HTTP**

本地单机用 stdio 最简单，跨进程通信没有网络开销。但如果你的 Agent 跑在容器里，MCP Server 在宿主机上，需要用 SSE 或 HTTP 传输。注意端口配置和 CORS 问题，MCP 的 HTTP 实现目前还是早期阶段，兼容性测试不能省。

## 可复用建议

- **把 MCP Server 做薄，把逻辑做厚**：MCP Server 只做协议翻译，真正的数据库访问、第三方 API 调用放在内层模块里。方便替换协议版本
- **工具命名带前缀**：比如 `db__query`、`github__create_issue`，避免多个 Server 暴露同名工具时冲突
- **每个工具都要有时间维度**：至少返回一个执行时间戳。排障时你会感谢我
- **用 mock server 做联调**：不要在开发时直接连生产环境的 MCP Server。写一个 `mock_server.py`，返回固定数据，先验证 Agent 的决策链路，再换真服务

## 总结

MCP 不是银弹，没有解决 Agent 的所有集成问题。它解决的是一个具体且重要的痛点：**让工具能力的描述、发现、调用这三件事有了统一的标准**。

对于 OpenClaw 这样的 Agent 框架，MCP 的价值在于：
- 插件与 Agent 解耦，同一个 Server 可以服务任何支持 MCP 的 Host
- 工具可以独立升级，不用改动 Agent 代码
- 你可以用最熟悉的语言写工具，协议层帮你兜底

现在 MCP 客户端配置项已经相当稳定，是开始实践的好时机。先用一个小工具跑通，再逐步替换掉现有的胶水代码。

---

