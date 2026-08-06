---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 31827
source: 综合讨论
publishedAt: 2026-08-06
---

## 背景：Agent 为什么非得“出去看看”？

如果我们把大模型看作一个博学但被困在房间里的专家，那么函数调用（function calling）就是递给它的一部电话。它能用，但每次打电话都得先查通讯录、现学拨号规则，挂了电话之后也不会主动整理笔记。随着 Agent 要打交道的工具越来越多——文件系统、数据库、API、浏览器、本地脚本——你会发现，自己不是在搭积木，而是在不断重复写“电话黄页”和“接线员培训手册”。

更麻烦的是，工具之间的上下文是割裂的。今天你可能需要先查数据库，再根据结果调用天气 API，最后让模型总结出一份差旅建议。每一步都需要开发者手动拼接上下文、处理错误，并且确保模型在 token 限制内记住所有关键信息。组合爆炸一出现，Agent 就变得脆弱，甚至直接把令牌窗口撑爆。

这就是 Agent 开发中长期存在、却一直被碎片化方案掩盖的问题：**上下文管理缺乏统一标准，工具发现和调用缺少通用协议**。

## MCP 到底是什么，解决了哪些具体问题？

Model Context Protocol（MCP）是 Anthropic 开源的一套标准化协议，用来连接大模型客户端和提供工具/数据的服务端。它并非新的模型能力，而是做了一件工程化的事：把模型对工具和外部数据的访问模式抽象成**客户端-服务器**的交互协议。

MCP 带来的核心改变有三点：

1. **上下文不再是“一次性快照”，而是动态资源**  
   传统做法里，外部数据需要在 prompt 组装时硬塞进去；一旦对话进入下一轮，这个上下文就可能被丢弃。MCP 定义了 `resources` 概念，服务端可以把文件内容、数据库视图、API 端点等作为可寻址资源暴露给模型。模型需要时直接引用，避免了大量重复的 token 消耗和手工拼接。

2. **工具调用不再需要每次手写 JSON Schema，且支持自动发现**  
   MCP 服务器启动时会声明自己提供了哪些 `tools`，并附带完整的输入结构描述。客户端（如 Claude Desktop、OpenClaw 这类运行核心）可以自动获取工具列表，无需开发者额外定义。你的 Agent 不用再背着笨重的“电话黄页”。

3. **交互提示和采样原语让流程更可控**  
   除了数据和工具，MCP 还设计了 `prompts` 和 `sampling` 原语，适合需要引导模型按模板输出，或者需要在服务端主动请求模型生成的场景。这为复杂工作流提供了更清晰的接口，而不是把大段系统指令藏在代码里。

一句话总结：**MCP 解决了 Agent 规模化对接外部世界时，上下文管理与工具集成“过于手工、过于脆弱”的工程痛点**。

## 如何快速上手：搭建一个 MCP 天气服务器

假设我们要让 Agent 通过 MCP 查询天气。服务端我们用 Python 写，客户端使用 Claude Desktop（同样适用于支持 MCP 的其他 Agent 环境）。

### 步骤 1：环境准备与安装
Python 版本需要 3.10 或以上，避免 asyncio 和 type hint 的兼容问题。创建虚拟环境后安装 MCP 官方包：
```bash
pip install mcp
```

### 步骤 2：编写 MCP 服务器
以下是一个极简版天气服务器，只暴露一个查询工具（示例使用模拟数据）：
```python
import asyncio
import json
from mcp.server import Server
from mcp.server.stdio import stdio_server

async def main():
    server = Server("weather-server")

    @server.list_tools()
    async def list_tools():
        return [{
            "name": "get_weather",
            "description": "获取指定城市的天气信息",
            "inputSchema": {
                "type": "object",
                "properties": {
                    "city": {"type": "string", "description": "城市名称，英文"}
                },
                "required": ["city"]
            }
        }]

    @server.call_tool()
    async def call_tool(name: str, arguments: dict):
        if name == "get_weather":
            city = arguments["city"]
            # 模拟天气数据，实际可调用 API
            return {
                "content": [{
                    "type": "text",
                    "text": json.dumps({
                        "city": city,
                        "temperature": "22°C",
                        "condition": "晴朗"
                    }, ensure_ascii=False)
                }]
            }

    async with stdio_server() as (read_stream, write_stream):
        await server.run(read_stream, write_stream,
                         server.create_initialization_options())

if __name__ == "__main__":
    asyncio.run(main())
```
代码逻辑很直接：定义工具列表，实现调用处理，通过 stdio 与客户端通信。

### 步骤 3：注册到 Claude Desktop
打开 Claude Desktop 的 `claude_desktop_config.json`（通常在 `~/Library/Application Support/Claude/` 或 `%APPDATA%\Claude\`），添加：
```json
{
  "mcpServers": {
    "weather": {
      "command": "python",
      "args": ["/absolute/path/to/weather_server.py"],
      "env": {}
    }
  }
}
```
重启 Claude Desktop，点击输入框旁的“插头”图标，就能看到“get_weather”工具已在列表中。

## 踩坑记录与可复用建议

在实际使用中，下面这些坑值得提前绕开：

- **路径和命令问题**：`command` 最好用绝对路径指向虚拟环境里的 `python`，并确保脚本的 shebang 正确。Windows 用户还可能会碰到控制台编码问题，建议在脚本第一行加上 `# -*- coding: utf-8 -*-`。
- **工具定义的 inputSchema 必须严格遵守 JSON Schema 规范**，缺少 `type` 或 `required` 字段往往会让客户端静默挂掉而不给明确报错。出错时检查客户端日志（Claude Desktop 在 `Help > Open MCP Logs`）。
- **服务器运行的模型是每会话独立**，不要在服务端保存全局状态，否则多会话并发会相互污染。需要状态时可用 `request_context` 中的 session 标识隔离。
- **调试利器**：MCP 提供了 `mcp dev` 命令行工具，可以模拟客户端调用，比反复启动 GUI 高效得多。
- **安全约束**：MCP 服务器通常运行在本地，但要注意工具权限最小化。例如文件系统操作工具，务必限定根目录，避免模型通过路径遍历访问敏感区域。

对于实际生产级 Agent（如 OpenClaw 社区常做的自动化编排），建议将 MCP 视为“工具接入层”的一部分，而不是全部。复杂状态管理、长短期记忆、多 Agent 协同仍需在上层封装。

## 总结

MCP 的价值不在于发明新概念，而在于把被无数 Agent 项目重复造过的轮子标准化。它让你能像管理微服务一样管理模型可用的工具和上下文资源。这个协议还很年轻，生态正在形成，但它的设计思路已经明确指向一个方向：让模型对外部世界的感知，不再是每次对话开始时的一次性注入，而是一种持续、可发现、可演进的连接。

对于尝鲜者，我的建议是：从替换一个你最常用的工具开始（比如文件检索或数据库查询），感受“工具自动可见”和“上下文资源化”带来的便利，再逐步扩展到复杂的自动化流程。毕竟，好的协议不是用来读的，是用来让工程变简单的。

---

