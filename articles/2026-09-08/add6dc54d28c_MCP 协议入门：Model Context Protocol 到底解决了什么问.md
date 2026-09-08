---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 36619
source: 综合讨论
publishedAt: 2026-09-08
---

## 背景

过去一年做 Agent 的人基本都在重复同一件事：给模型接工具。查数据库要写一套 function calling，读 GitHub 要再写一套，接公司内部 API 又是一套。每接一个工具，就要在宿主应用里写一份专属的描述、参数定义、鉴权和错误处理。工具本身不难，难的是这种集成没有任何标准可言。

MCP（Model Context Protocol）就是冲着这个来的。它由 Anthropic 在 2024 年底开源，采用 JSON-RPC 2.0 作为通信格式，目前已被多家主流模型厂商和 Agent 框架采纳。一句话概括：它给「模型应用如何连接外部工具和数据」定了统一的接口规范。

## 它解决了什么问题

本质上是把 **M×N 变成 M+N**。没有 MCP 时，M 个 AI 应用对接 N 个工具，最坏要写 M×N 份适配代码；有了 MCP，应用侧实现一次 client，工具侧实现一次 server，两边按协议握手即可互相发现能力。适配层从「给某个应用定制」变成「给整个生态通用」。

对 OpenClaw / Agent 用户来说还有一层实际意义：插件体系和 MCP 不冲突。插件适合承载与宿主深度耦合的逻辑，MCP server 适合封装通用的、跨场景复用的能力——你今天写的数据查询 server，换个 Agent 框架照样能用。

协议本身不复杂，三个核心原语：

- **Tools**：模型可主动调用的动作（执行侧）
- **Resources**：应用可读取的数据（上下文侧）
- **Prompts**：预设的提示模板（用户触发侧）

传输层两种常用模式：本地进程走 stdio，远程服务走 Streamable HTTP。

## 上手步骤

用 Python SDK 写一个最小 MCP server，十几行：

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("weather")

@mcp.tool()
def get_weather(city: str) -> str:
    """查询指定城市的实时天气，参数 city 为城市名。"""
    # 这里调用真实 API
    return f"{city}: 晴 26°C"

mcp.run(transport="stdio")
```

在客户端（桌面端、IDE 或你自己的 Agent runtime）里注册：

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

运行流程是：client 启动 server 进程 → initialize 握手协商版本和能力 → client 拉取工具列表 → 模型决定调用时，client 发 `tools/call`，server 返回结果。建议先用官方的 MCP Inspector 手动跑一遍，确认工具列表和调用行为符合预期，再接入 Agent 流程。

## 踩坑点

1. **stdio 模式下 stdout 是协议通道。** 任何 `print` 调试日志都会污染 JSON-RPC 帧，导致 client 解析失败。日志一律走 stderr 或写文件。
2. **工具描述就是给模型看的 API 文档。** 描述含糊，模型要么不调用，要么传错参数。参数 schema 和 docstring 值得认真写，这是回报率最高的投入。
3. **不要无脑 auto-approve。** 工具返回的外部内容可能携带注入指令，写操作（删数据、发消息、花钱）务必加人工确认或白名单。
4. **工具数量失控。** 挂几十个工具后，上下文膨胀、模型选择准确率明显下降。按领域拆 server，单个 server 保持精简。
5. **版本兼容。** 协议仍在演进（例如 transport 从 HTTP+SSE 改为 Streamable HTTP），自建 client/server 时注意握手阶段的版本协商，别硬编码。

## 可复用建议

- 把 MCP server 当作已有 API 的**薄适配层**，业务逻辑留在服务里，server 只做协议转换；
- 一个领域一个 server，工具保持小而正交，命名加域前缀避免冲突；
- 团队内固化一个脚手架模板（stderr 日志、统一错误处理、最小权限），新工具照着填；
- 上线前用 Inspector 过一遍边界情况：空参数、超时、异常返回。

## 总结

MCP 没有黑科技，它的价值就是标准化：把重复的集成劳动收敛成一套协议，让工具写一次、处处可接。但它不替你解决鉴权、安全和工具设计质量——这些依然是你的工程责任。务实的用法是：把它当接口规范用，小步接入，先跑通一个 stdio server，再考虑远程部署和更细的权限控制。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/c1556ada336d30af.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/a00a1b2dc25d875d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/11caa0ead5f08aa3.png)

