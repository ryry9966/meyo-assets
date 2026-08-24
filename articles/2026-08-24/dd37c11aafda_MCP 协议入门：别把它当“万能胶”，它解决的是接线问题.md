---
title: MCP 协议入门：别把它当“万能胶”，它解决的是接线问题
feedId: 34535
source: 综合讨论
publishedAt: 2026-08-24
---

MCP 在 Agent 圈子里已经不是新名词。但对实际写工具、接自动化的人来说，最容易把它理解成一种 RPC 或插件标准：它不产生智能，只负责把模型和外部能力之间的线接稳。本文从工程角度梳理它到底解决了什么、怎么接、哪里会踩坑。

## 背景：工具接入为什么这么烦

在 MCP 之前，给 Agent 接外部能力通常有几种做法：

- 直接调用模型自带的 function calling，每个模型、每个客户端都要写一套 schema；
- 把文件内容、数据库查询结果手动塞进 system prompt，又长又容易爆上下文；
- 为每个 Agent 框架写一遍工具适配层，换框架约等于重写。

结果就是胶水代码越来越多，工具、资源、提示三件事混在一起，难以复用。MCP 想做的是把这层“连接”标准化：工具、资源、提示都通过统一的 client-server 协议暴露。

## MCP 到底解决了什么问题

MCP 解决的核心问题可以拆成三类：

1. **工具调用标准化**：通过 `tools/list`、`tools/call` 等 JSON-RPC 消息交互，客户端不用关心后端是 Python、Node 还是 CLI。
2. **上下文资源访问**：`resources/list`、`resources/read` 提供文件、数据库记录、文档等，模型按需读取，不用提前拼进 prompt。
3. **多客户端复用**：一个 MCP server 可以同时接给 OpenClaw、桌面客户端、自研 Agent 等，减少重复开发。

它不解决业务逻辑正确性、权限隔离、模型幻觉、工具执行副作用，也不适合高频、低延迟的内部函数调用。MCP 更适合跨进程、跨语言、需要复用的外部能力封装。

## 做法/步骤：从一个最小 MCP server 开始

以 Python 为例，使用官方 SDK 里的 `FastMCP` 写一个 echo 工具。

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("demo")

@mcp.tool()
def echo(text: str) -> str:
    """Return the input text. Use when the user wants to test tool calling."""
    return text

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

保存为 `demo_server.py`，然后在 OpenClaw 的 MCP 配置中接入：

```json
{
  "mcpServers": {
    "demo": {
      "command": "python",
      "args": ["demo_server.py"],
      "cwd": "/absolute/path/to/project"
    }
  }
}
```

调试时先用官方 inspector 连一次：

```bash
npx @modelcontextprotocol/inspector python demo_server.py
```

确认 `tools/list` 能返回工具、`tools/call` 能正常执行后，再接到 OpenClaw 里跑真实会话。这样能排除一半以上“配置没问题但工具不出现”的情况。

## 踩坑点

- **stdio 工作目录**：配置里用相对路径时，找不到命令十有八九是 `cwd` 不对。直接写绝对路径最省事。
- **工具描述太模糊**：模型不知道何时调用。描述要写清楚“什么时候用、输入输出是什么、有什么限制”，不要只写一句“执行工具”。
- **返回值过大**：直接把整个文件或数据库结果返回，会撑爆上下文。工具内部要截断、分页或返回摘要。
- **权限边界**：MCP server 具有执行权限，不要随便接来源不明的 server。本地运行也要限制可访问路径和命令白名单。
- **能力支持不一致**：`resources`、`prompts` 等能力在不同客户端支持程度不同，接入前先确认目标客户端是否支持，不要默认所有能力都能用。

## 可复用建议

- 优先封装已有 CLI、HTTP API 为 MCP，而不是从零写复杂业务。
- 先只读后写入：先暴露查询类工具，稳定后再加写操作。
- 用 JSON Schema 严格定义参数，避免模型自由发挥。
- 工具返回统一用 `{content: [{type: "text", text: ...}]}`，出错时标记 `isError`。
- 每个 MCP server 写一个 README，说明可用工具、权限边界和测试命令。
- 按业务域拆分多个小型 MCP server，不要做一个巨型 server。

## 总结

MCP 解决的是 Agent 与外部能力之间的“接线标准化”。它不会让你的模型更聪明，但能让你少写很多胶水代码，让工具、资源、提示在不同客户端间复用。入门时从一个小而明确的只读工具开始，跑通 stdio 调试和 OpenClaw 接入，比一开始就上复杂资源映射更实际。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/74ecbc9cb5a9def7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/18240b2c6b4670b9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/760a57a5af007bbf.png)

