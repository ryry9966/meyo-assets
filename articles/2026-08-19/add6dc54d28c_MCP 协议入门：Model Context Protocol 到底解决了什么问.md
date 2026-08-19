---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 33851
source: 综合讨论
publishedAt: 2026-08-19
---

## 背景：Agent 工具接入的碎片化

在 OpenClaw 这类 Agent 框架里，给模型接工具一直是个体力活。早期每个平台都有自己的 Function Calling 格式：OpenAI 一套，Claude 一套，开源框架又各自定义插件接口。你给 Claude 写了一个查天气的工具，迁移到另一个 Agent 客户端时，适配层几乎要重写。

MCP（Model Context Protocol）的出现，本质上是把这层“工具-模型”的连接标准化。它定义了一套 JSON-RPC 消息格式，让模型客户端（Host）可以通过统一的协议调用外部工具、读取资源、使用提示模板。一个 MCP Server 写好后，理论上可以被任何支持 MCP 的客户端加载，不需要关心底层是 Claude、OpenAI 还是 OpenClaw 自己的 Agent 循环。

## 它到底解决了什么问题

不是“让模型变强”，而是**减少胶水代码**。具体来说：

1. **一次编写，多处复用**：文件系统、数据库、浏览器自动化这些通用能力，社区已经有现成的 MCP Server，直接配置就能用。
2. **统一工具描述格式**：工具的输入输出 schema、描述文本都有规范，客户端可以自动生成工具列表，降低模型误调用概率。
3. **解耦工具实现与模型调用**：MCP Server 独立进程运行，工具逻辑不侵入 Agent 核心代码，升级、重启、权限隔离都更干净。

## 最小实践：写一个时间查询 MCP Server

下面是一个最简 Python MCP Server，暴露一个 `get_current_time` 工具：

```python
# server.py
import asyncio
import datetime
from mcp.server import Server
from mcp.server.stdio import stdio_server

app = Server("time-server")

@app.tool()
async def get_current_time() -> str:
    """返回当前 ISO 8601 格式时间"""
    return datetime.datetime.now().isoformat()

async def main():
    async with stdio_server() as (read_stream, write_stream):
        await app.run(read_stream, write_stream)

if __name__ == "__main__":
    asyncio.run(main())
```

安装依赖后，在 OpenClaw 的 MCP 配置中加入：

```json
{
  "mcpServers": {
    "time": {
      "command": "python",
      "args": ["/absolute/path/to/server.py"]
    }
  }
}
```

重启 OpenClaw 后，Agent 的工具列表里会出现 `get_current_time`，模型可以直接调用并拿到结果。整个过程没有修改任何 Agent 主流程代码。

## 踩坑点

1. **stdio 与 SSE 的选择**  
   stdio 适合本地进程，通过标准输入输出通信；SSE/HTTP 适合远程 Server。如果你把 MCP Server 部署在容器或另一台机器，别用 stdio，客户端会一直等待本地进程启动而超时。

2. **工具描述要写清楚**  
   MCP 工具的描述直接进入模型的上下文。如果描述模糊、参数含义不明，模型很可能传错参数或调用错误工具。不要只写函数名，要写清楚“做什么、参数是什么、返回什么”。

3. **权限边界**  
   MCP Server 拥有启动进程的完整权限。用文件系统类 Server 时，务必限制根目录，例如只允许访问 `/data/agent_workspace`，而不是 `~`。否则一次误调用可能删掉重要文件。

4. **调试困难时先看协议层**  
   如果工具注册了但模型不调用，先用 `npx @modelcontextprotocol/inspector` 单独测试 Server，确认 JSON-RPC 消息正常。很多时候是 Server 启动失败或返回了非标准错误，客户端静默忽略了。

## 可复用建议

- **优先复用现成 Server**：`mcp-server-filesystem`、`mcp-server-github`、`mcp-server-puppeteer` 这些社区项目已经覆盖了大量通用场景。先看有没有能直接用的，再考虑自己写。
- **工具粒度拆小**：一个工具只做一件事。比如“读取文件”和“写入文件”分开，不要让一个 `file_operation` 工具同时处理读、写、删除。模型更容易选择正确工具。
- **敏感信息用环境变量**：数据库连接串、API Key 不要硬编码在 Server 代码里。通过环境变量传入，配置在 OpenClaw 的 `env` 字段中。
- **控制工具数量**：一次加载太多工具会干扰模型决策。按场景分组，例如“开发环境”“数据查询”“浏览器自动化”，只加载当前任务需要的 MCP Server。

## 总结

MCP 解决的是工具接入层的标准化问题，它让 Agent 生态从“每家都写适配器”走向“一个协议多处复用”。但它不是银弹：模型推理能力、任务规划、记忆管理这些核心问题，MCP 并不直接解决。务实的态度是：能用现成 MCP Server 就不造轮子，必须自己写时保持工具简单、描述清晰、权限最小化。在 OpenClaw 里合理配置 MCP，确实能把大量胶水代码从维护清单里划掉。

---

