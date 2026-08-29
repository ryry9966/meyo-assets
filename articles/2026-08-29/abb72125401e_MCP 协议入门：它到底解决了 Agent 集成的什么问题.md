---
title: MCP 协议入门：它到底解决了 Agent 集成的什么问题
feedId: 35230
source: 综合讨论
publishedAt: 2026-08-29
---

## 背景：Agent 工具集成的胶水代码越写越多

在 OpenClaw 这类 Agent 框架里，模型真正有用的场景很少是“只聊天”。更多时候，它需要查数据库、调内部 API、读文件、发消息、操作浏览器。每接一个新系统，我们通常要写一个插件或适配器：把业务接口封装成模型能理解的函数描述，再处理参数解析、错误返回、权限控制。

问题在于，这些胶水代码和具体 Agent 框架耦合。同一个工具，在 A 框架里写一遍，换到 B 框架又要重写。工具定义分散在不同配置文件、不同语言、不同进程里，维护成本随接入数量指数上升。MCP 想解决的就是这个“N 个工具 × M 个客户端”的重复适配问题。

## 它到底标准化了什么

MCP（Model Context Protocol）不发明新工具，它把工具接入方式标准化。类似 USB-C 不发明外设，但统一了接口形状。MCP 基于 JSON-RPC 2.0，定义了三个核心原语：

- **Tools**：模型可以调用的操作，比如查询订单、发送消息。
- **Resources**：只读上下文数据，比如知识库文档、数据库 schema。
- **Prompts**：可复用的提示模板，比如“帮我总结这份日志”。

客户端（例如 OpenClaw、Claude Desktop、自研 Agent）只需要实现 MCP client，就能访问任何 MCP server 暴露的能力。服务端开发者不再关心模型是 GPT、Claude 还是本地模型，只负责把业务能力按 MCP 规范暴露出来。

## 最小实践：跑通一个 MCP Server

以 Python 为例，环境用 Python 3.10+ 和 uv。

**1. 安装 SDK**

```bash
uv add mcp
```

**2. 写一个最小 server**

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("demo")

@mcp.tool()
def add(a: int, b: int) -> int:
    """返回两个整数之和"""
    return a + b

if __name__ == "__main__":
    mcp.run()
```

保存为 `server.py`。这个 server 通过 stdio 与客户端通信，暴露了一个 `add` 工具。

**3. 在客户端配置**

以 Claude Desktop 或 OpenClaw 的 MCP 配置为例：

```json
{
  "mcpServers": {
    "demo": {
      "command": "python",
      "args": ["/absolute/path/to/server.py"]
    }
  }
}
```

**4. 验证**

可以用 MCP Inspector 快速调试：

```bash
npx @modelcontextprotocol/inspector python /absolute/path/to/server.py
```

看到工具列表里出现 `add`，并且能成功调用，就说明最小闭环通了。再把 `add` 替换成真实业务操作即可。

## 踩坑点

**传输层选择**  
stdio 适合本地单机工具；SSE/HTTP 适合远程服务。但远程模式没有内置鉴权，裸暴露公网非常危险。要么放在内网，要么在反向代理层加认证。

**工具描述就是 prompt**  
模型靠 `description` 和参数 schema 判断何时调用、怎么传参。描述写“传入 a 和 b”但不说清楚用途、返回格式、副作用，模型就会乱调。参数尽量用 JSON Schema 约束类型、枚举、默认值和 `required` 字段。

**Resources 与 Tools 不要混用**  
只读上下文用 Resources，有副作用的操作用 Tools。模型不会天然知道一个工具会不会扣款、发消息，必须在描述里明确告知“该操作会产生副作用”。

**版本与路径问题**  
MCP SDK 更新较快，不同客户端支持的特性有差异。先固定 SDK 版本再升级。客户端启动 server 时的工作目录可能不是你期望的路径，配置里使用绝对路径，并通过 `env` 字段传环境变量。

**错误处理**  
工具内部异常要捕获并返回结构化错误，不要让进程直接崩溃。stdio 模式下，任何非 JSON-RPC 输出都会污染通信，调试日志只能写到 stderr 或文件。

## 可复用建议

1. **先跑通 add/echo，再接真实工具**。不要一上来就接数据库和消息队列。
2. **一个 server 只做一类事**。工具数量过多会降低模型选择准确率，建议按领域拆分。
3. **给每个工具写清楚用途、参数、返回值、副作用**。这是最容易被忽略的工程文档。
4. **把 MCP server 当独立进程维护**。相比嵌入式插件，独立进程更容易隔离故障、单独重启和升级。
5. **权限最小化**。给 server 单独的数据库账号、API token 和命名空间，避免模型误调用造成大范围影响。
6. **使用容器或虚拟环境固定依赖**。防止系统 Python 环境变化导致 server 启动失败。

## 总结

MCP 解决的核心问题不是“让模型变聪明”，而是把工具接入从框架绑定的胶水代码中解放出来。对 OpenClaw 用户而言，它更像一个标准化的插件接口：你写一次 MCP server，多个 Agent 客户端都能复用。

但它不是银弹。安全性、模型调用准确性、工具描述质量、版本兼容，这些仍然需要工程化地处理。建议先在本地跑通官方 quickstart，再评估是否引入到实际业务链路。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/26cc8cc9688914b5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/fc17b028df374305.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/1c94f0a51959a30d.png)

