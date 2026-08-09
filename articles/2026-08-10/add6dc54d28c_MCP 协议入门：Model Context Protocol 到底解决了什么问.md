---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 32286
source: 综合讨论
publishedAt: 2026-08-10
---

## 为什么需要 MCP？

用过 OpenClaw 做自动化或 Agent 开发的同学一定有这样的经历：为大模型接入一个简单的外部数据源（比如读取本地文件或查询数据库），就得写一整套胶水代码。先解析用户意图，再手动拼接到工具调用的函数签名，返回值又要裁剪成模型能理解的上下文，每一步都在重复造轮子。工具多了之后，上下文膨胀、权限失控、插件难以复用的问题便接踵而至。

这套“手搓”流程的根源在于：模型与外部世界之间缺少一个**统一的交互协议**。Anthropic 在 2024 年底开源的 Model Context Protocol (MCP) 正是试图解决这一问题的标准化方案。它不是又一个“LLM 框架”，而是一组轻量的通信规范，定义了 AI 应用如何安全、可发现地访问外部资源和工具，类似于“USB-C 之于外设”。

## MCP 的核心设计：资源、工具与提示

MCP 基于客户端-服务端架构，运行在 JSON-RPC 2.0 之上。它抽象出三个核心原语：

- **资源（Resources）**：可被模型读取的数据，比如文件内容、数据库记录、API 响应。每个资源通过 URI 标识（如 `file:///path/to/doc`），模型可以像访问“文件系统”一样获取上下文。
- **工具（Tools）**：模型可调用的函数。服务端暴露具名工具及 JSON Schema 描述的参数，客户端（Agent）让模型选择工具并生成调用请求。
- **提示（Prompts）**：预置的 prompt 模板，用于引导交互或复用固定工作流。

简单来说，MCP 将“如何暴露能力”和“如何消费能力”解耦。服务端只管实现 `list_resources`、`read_resource`、`list_tools`、`call_tool` 等方法，客户端通过标准协议发现并调用，无需关心内部细节。

## 实操：用 Python 搭建一个 MCP 文件服务器

我们以一个最小化场景为例：让 Agent 通过 MCP 读取本机指定目录下的文本文件，并返回文件名列表和内容。

### 1. 环境准备
使用官方 Python SDK `mcp`。需要 Python 3.10+。
```bash
pip install mcp
```

### 2. 编写服务端
创建一个 `server.py`，定义一个资源列表：扫描给定目录下的 `.txt` 文件，每个文件映射为一个资源。
```python
import os
from mcp.server import Server, NotificationOptions
from mcp.server.models import InitializationCapabilities
from mcp.server.stdio import stdio_server
from mcp.types import Resource, Tool, TextContent

server = Server("file-server")

@server.list_resources()
async def list_resources():
    base_dir = os.getenv("MCP_ROOT_DIR", ".")
    files = [f for f in os.listdir(base_dir) if f.endswith(".txt")]
    return [
        Resource(
            uri=f"file://{base_dir}/{f}",
            name=f,
            description=f"Text file: {f}"
        )
        for f in files
    ]

@server.read_resource()
async def read_resource(uri: str):
    path = uri.replace("file://", "")
    with open(path, "r") as f:
        return [TextContent(type="text", text=f.read())]

async def main():
    async with stdio_server() as (read_stream, write_stream):
        await server.run(
            read_stream,
            write_stream,
            InitializationCapabilities(
                sampling={},
                experimental={},
                roots={}
            ),
            notification_options=NotificationOptions()
        )

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

这里使用 `list_resources` 暴露文件列表，`read_resource` 按 URI 读取内容。传输方式选择 stdio，这是 MCP 服务最轻量的部署方式，适合本地 Agent 调用。

### 3. 在 Agent 客户端中集成
以 Claude Desktop 为例（OpenClaw 的 Agent 同样可以基于 MCP 客户端实现），在配置文件中注册这个服务：
```json
{
  "mcpServers": {
    "my-file-server": {
      "command": "python",
      "args": ["/absolute/path/to/server.py"],
      "env": {
        "MCP_ROOT_DIR": "/home/user/docs"
      }
    }
  }
}
```
重启客户端后，模型就能自动发现并读取该目录下的所有 txt 文件了。在 OpenClaw 中，你可以用类似的思路编写 MCP 客户端适配器，或直接复用社区提供的 MCP 连接器，让工作流动态挂载任意外部能力。

## 踩坑点：从协议到落地

1. **传输层选择误区**  
   MCP 同时支持 stdio 和 HTTP+SSE 两种传输。stdio 适合本地开发，简单无依赖，但不适用远程服务；HTTP 方式需要自己处理认证和消息流，早期版本的 Python SDK 对 HTTP 支持还不稳定。若在 Docker 或远程服务器部署，建议先评估客户端是否支持 SSE，否则改用 stdio + 进程管理。

2. **资源 URI 的规范化**  
   URI 是资源的唯一标识，格式必须与客户端解析逻辑一致。比如文件资源建议用 `file://` 前缀，API 资源用 `https://`，避免模型因 URI 不规范而拒绝调用。实践中，URI 中包含特殊字符时需要 percent-encode，否则 JSON-RPC 会解析失败。

3. **工具参数描述必须精确**  
   模型完全依赖 Schema 描述来决定何时以及如何调用工具。如果 description 太模糊，模型可能从不调用；如果参数缺少 `enum` 或 `default`，容易产生幻觉参数。强烈建议为每个工具补充充分的使用示例到 description 字段。

4. **权限隔离不足**  
   服务端暴露的资源通常没有细粒度权限控制。一旦 Agent 能列出目录，它就可能读任何目标文件。生产环境务必限定 MCP 服务的可访问范围（如通过容器挂载只读目录），并在客户端层增加确认机制。

## 工程化复用建议

- **将通用能力封装为独立 MCP 服务**：比如文件系统、数据库查询、HTTP API 网关、代码执行沙箱，均可做成可拔插的 MCP 服务，通过配置注入不同 Agent。
- **善用社区现成服务器**：官方提供了 filesystem、github、postgres、puppeteer 等实现，OpenClaw 项目可直接拉取作为 sidecar 进程运行，避免重复开发。
- **构建资源发现链**：在 OpenClaw 的插件系统中，让 Agent 在启动时扫描已连接的 MCP 服务列表，动态获取工具清单并注入到系统提示中，实现“发现即所用”。
- **日志与排障**：MCP 基于标准输入输出或 HTTP 流，所有交互都是 JSON-RPC 消息。建议在 Agent 层记录完整请求/响应，便于调试工具调用错误。

## 总结

MCP 解决的不是“模型能不能调用工具”的问题，而是“如何以一种优雅、可复用、可发现的方式标准化这种调用”。它让外部数据接入从手工作坊式的胶水代码，进化到了即插即用的协议层。对于 OpenClaw 社区中频繁与各种系统打交道的 Agent 开发者来说，拥抱 MCP 意味着工具链的可组合性将迈上一个台阶，也意味着你的自动化工作流可以像搭积木一样自由拼接外部能力。

下一步可以尝试：将一个已有的自定义工具改造为 MCP 服务，再用 OpenClaw 的插件机制加载，感受标准协议带来的解放感。

---

