---
title: MCP 协议入门：它到底解决了 Agent 工具接入的什么问题？
feedId: 35233
source: 综合讨论
publishedAt: 2026-08-29
---

## 一、先看没有 MCP 时，Agent 接工具是什么样

在 OpenClaw 这类 Agent 项目里，最麻烦的不是模型本身，而是让模型稳定地调用外部能力。早期常见做法是：每个工具写一个 Python 函数，再手写 JSON Schema，再把结果拼回 prompt。数据源多了以后，会出现几类问题：

- 每接一个外部系统，就要写一套“发现—调用—序列化—错误处理”的胶水代码；
- 同一个工具在不同 Agent 项目里无法复用，换框架要重写；
- 上下文里塞工具描述、资源内容、执行结果的方式各不相同，参数一多，模型很容易选错工具。

MCP（Model Context Protocol）解决的不是“让模型变聪明”，而是把 Agent 与外部工具/资源之间的接口标准化。它基于 JSON-RPC 2.0，定义了客户端和服务端之间如何发现能力、调用工具、读取资源。对工程来说，最大的收益是：工具接入从“项目内代码”变成“可复用服务”。

## 二、MCP 到底标准化了什么

可以把 MCP 理解成“Agent 界的 USB-C”。它主要抽象了三类能力：

1. **Tools**：可被模型调用的函数，有名字、描述、JSON Schema 参数；
2. **Resources**：可被模型读取的数据，比如文件、数据库查询结果、API 返回；
3. **Prompts**：可复用的提示词模板。

服务端暴露这些能力，客户端通过 `initialize`、`tools/list`、`tools/call` 等 JSON-RPC 消息交互。一个 MCP Server 可以用 stdio 跑在本地，也可以用 HTTP/SSE 暴露成远程服务。这样 OpenClaw 接入一个外部系统时，不再需要改内核，只需要配置一个 MCP Server 命令或 URL。

## 三、一个最小可复现的 MCP Server

以 Python 为例，先用官方 SDK 写一个加法工具。项目里安装：

```bash
pip install "mcp[cli]"
```

创建 `server.py`：

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("demo")

@mcp.tool()
def add(a: float, b: float) -> float:
    """Return the sum of two numbers."""
    return a + b

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

在 OpenClaw 或支持 MCP 的客户端里，新增一个 Server 配置，类型选 `stdio`，命令填：

```bash
python /path/to/server.py
```

启动后客户端会先发 `initialize`，然后列出 `add` 工具。模型问“帮我算 3 加 5”时，会触发一次 `tools/call`，服务端返回 `8`。

在本机调试时，可以直接用一个简单的 MCP 客户端脚本验证，不依赖 OpenClaw：

```python
import asyncio
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def main():
    server = StdioServerParameters(command="python", args=["server.py"])
    async with stdio_client(server) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()
            tools = await session.list_tools()
            print(tools)

asyncio.run(main())
```

这一步能跑通，后续再接 OpenClaw 插件层会少很多干扰。

## 四、实际踩坑点

1. **stdio 被调试输出污染**  
   Server 里不要用 `print()` 打日志。stdout 是协议通道，打印任何东西都会破坏 JSON-RPC 解析。日志统一走 `stderr`，或者用 `mcp` 的日志接口。

2. **工具描述写得太随意**  
   模型非常依赖工具名和 description。`get_data` 这种名字基本会被调用错。建议工具名用动词开头，描述写清“输入什么、输出什么、什么情况下用”。参数描述也要具体，不要只写 `id`，写 `user_id: The numeric user ID from the auth system`。

3. **大结果撑爆上下文或超时**  
   MCP 不限制返回大小，但模型上下文有限。一个数据库查询返回几万行，很容易把任务搞挂。服务端应该限制返回行数，或先返回摘要，再提供进一步查询的工具。

4. **版本兼容问题**  
   MCP SDK 0.x 变化较快，不同客户端支持的协议版本可能不一致。接上线前先在本地验证 `initialize` 和 `tools/call`，确认服务端声明的能力与客户端实际调用一致。

5. **远程 Server 的安全边界**  
   如果把 MCP Server 暴露成 HTTP/SSE，相当于对外提供了一个可被模型驱动的 API。必须做鉴权，并且不要在 Server 里直接包装无限制的 shell 或数据库连接。

## 五、可复用建议

- **工具粒度尽量薄**：一个工具只做一件事，组合逻辑交给 Agent 或外部流程，比写一个“万能工具”更可控；
- **把 MCP Server 当服务治理**：给每个 Server 写 README、配置模板、健康检查命令，避免插件一多就不可维护；
- **错误也要结构化**：不要返回裸异常字符串，尽量返回 `{"ok": false, "error": "...", "hint": "..."}`，模型能更好恢复；
- **配置与代码分离**：API key、路径、数据库连接放环境变量，MCP Server 配置里不要硬编码敏感信息；
- **先小范围验证**：新工具先在命令行客户端或最小脚本里跑通，再挂到 OpenClaw 的 Agent 流程里。

## 六、总结

MCP 的价值不在于它多新，而在于它把 Agent 接入外部能力这件事从“项目内胶水代码”下沉成“可复用服务”。对 OpenClaw 用户来说，合理使用 MCP 能减少重复适配，让插件和自动化更清晰。但它也不能解决所有问题：模型仍然会选错工具、上下文仍然有限、Server 仍然需要治理。把它当成一个工程接口来用，而不是当成万能连接器，会更实际。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/72b8297b0bfa793d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/cfac0ed36ad9b7fc.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/e634f901a29164e4.png)

