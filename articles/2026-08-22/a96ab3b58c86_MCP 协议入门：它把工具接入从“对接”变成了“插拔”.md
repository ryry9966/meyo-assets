---
title: MCP 协议入门：它把工具接入从“对接”变成了“插拔”
feedId: 34142
source: 综合讨论
publishedAt: 2026-08-22
---

## 背景

如果你在 OpenClaw 这类 Agent 宿主里接过工具，大概率经历过：Slack 机器人要写一套工具接入，本地 CLI 要写另一套，换到另一个模型或框架，又要重写工具描述、处理 JSON Schema、管理连接生命周期。模型本身没变，变的全是胶水代码。

Model Context Protocol (MCP) 就是为解决这个重复劳动提出的。它由 Anthropic 在 2024 年底开源，核心目标很朴素：给 LLM 应用定义一套统一的“客户端-服务器”协议，让工具、资源和提示可以通过标准接口暴露，任何兼容 MCP 的客户端都能直接消费。

一句话：MCP 不是让模型更强，而是让工具连接更标准化，类似 USB-C 之于外设。

## 它到底解决了什么问题

1. **N×M 集成爆炸**。传统做法是每个 Agent 框架都要为每个数据源/工具写适配器。MCP 把 M 个工具暴露为 MCP server，N 个客户端只需实现一次 MCP client 协议。
2. **上下文传递不统一**。工具描述、参数 schema、返回内容格式过去各写各的，经常出现模型理解偏差。MCP 统一了 tool 的声明结构：name、description、inputSchema。
3. **能力边界不清晰**。MCP 把能力拆成 tools（可执行操作）、resources（可读上下文，如文件、数据库记录）、prompts（可复用的提示模板）。这让模型知道什么时候该调工具，什么时候该读资源，而不是把所有内容硬塞进 system prompt。
4. **连接生命周期标准化**。初始化、能力协商、调用、关闭都有明确流程，客户端可以动态发现服务端提供什么，而不是硬编码。

## 最小实践：连接一个 MCP Server

以 Python SDK（`pip install mcp`）为例，构建一个获取天气的工具：

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("weather")

@mcp.tool()
def get_weather(city: str) -> str:
    """Get current weather for a city. Use when user asks about weather."""
    return f"{city}: 26C, cloudy"

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

在 OpenClaw 或任意支持 MCP 的客户端里，只需配置启动命令：

```json
{
  "mcpServers": {
    "weather": {
      "command": "python",
      "args": ["weather_server.py"]
    }
  }
}
```

客户端启动后会自动通过 stdio 连接、列出工具、按需调用。工具描述和 schema 与客户端解耦，服务端可以独立迭代。

## 踩坑点

1. **stdio 日志会污染协议通道**。MCP stdio 传输中，stdout 是协议数据，日志必须写到 stderr，否则客户端解析直接失败。排查半天才发现是 `print` 惹的祸。
2. **工具返回内容过大**。模型上下文窗口有限，如果一次返回整个数据库表或长文件，后续推理质量会明显下降。建议工具返回精简结果，并用 resources 提供大块上下文，让模型按需读取。
3. **description 写得太随意**。MCP 只解决传输格式，不解决语义理解。如果工具描述只有“get data”，模型会乱调。要写清楚“何时用、输入是什么、返回什么、有什么限制”。
4. **协议版本兼容**。不同 SDK 版本的 protocolVersion 可能不一致，初始化协商失败时优先检查客户端和服务端是否都支持同一版本，别急着改业务逻辑。
5. **SSE/HTTP 与 stdio 选择**。本地进程用 stdio 简单；远程部署或多客户端共享时，SSE/HTTP 更合适。混用时会遇到连接生命周期不一致的问题。

## 可复用建议

- 把每个 MCP server 当作一个 API 边界，不要做成“万能工具包”。一个服务聚焦一组相关能力，比如天气、日历、数据库查询。
- 工具和资源分开：工具是动作，资源是上下文。不要让工具返回大段上下文。
- 用 MCP Inspector 调试（`npx @modelcontextprotocol/inspector`），它能可视化列出工具、手动调用、查看 schema，比盲调客户端高效。
- 固定 SDK 版本并写进 README，避免团队内不同版本导致协议协商失败。
- 测试时先脱离模型，直接用 Inspector 调工具验证返回结构；再接入 Agent 测试模型调用准确率。

## 总结

MCP 解决的既不是模型智商问题，也不是 prompt 技巧问题，而是工具连接的工程成本。它把“对接”变成“插拔”，让 Agent 宿主、工具开发、上下文获取各层可以独立演进。落地时真正决定效果的，仍然是工具设计是否清晰、返回是否克制、边界是否合理。先从小型 MCP server 开始，把接口和描述打磨清楚，比一次性接入一堆工具更有价值。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/641ae29826b152b1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/90faca9c5d1934f5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/559fc2220222b4f5.png)

