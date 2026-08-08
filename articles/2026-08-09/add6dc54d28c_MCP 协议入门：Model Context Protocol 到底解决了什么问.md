---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 32174
source: 综合讨论
publishedAt: 2026-08-09
---

## 为什么需要 MCP

如果你正在使用 OpenClaw 构建 Agent，大概率已经尝过“手动把工具塞进对话”的滋味：每个外部能力（操作文件、查数据库、调 API）都要写一段专用的 function calling 代码，还要拼命维护工具描述、参数 schema、返回值格式——模型换个版本可能又得重新调试。这不是设计问题，而是行业缺少一个**模型与外设之间的通用交互标准**。

这就是 Model Context Protocol（MCP）的切入点。它由 Anthropic 提出，目标是为大模型提供一种标准化的、可插拔的外部上下文访问方式，把工具调用、资源获取与提示模板统一到同一套协议下。

一句话概括：**MCP 让 Agent 与任意外部工具/数据源之间的连接，像 USB 外设一样即插即用。**

## 无 MCP 时代的痛

以 OpenClaw 用户最熟悉的场景为例：你想让 Agent 读取本地文件，大概率会自己写一个 `read_file` 工具，通过 OpenAI 兼容的 function calling 传入模型。接下来还想加网络搜索、数据库查询，就要继续写 `search_web`、`query_db`，每个工具都要处理不同的鉴权、序列化、错误码。更麻烦的是，如果某天你想让同一个 Agent 在另一个客户端（比如 Claude Desktop）中使用，这些工具代码很难直接复用——因为每个客户端的工具注册方式完全不同。

于是你陷入两难：要么为每个平台重复实现相同的工具逻辑，要么把你的 Agent 和某个框架牢牢绑死。

MCP 试图从协议层面解决这个问题：**把工具的定义、发现与调用放在一个独立的服务进程中**，通过标准化传输通道（如 Stdio、HTTP+SSE）暴露给任意支持 MCP 的客户端。这样，工具开发者和 Agent 使用者就可以解耦。

## MCP 是如何工作的

MCP 采用经典的 **客户端-服务器架构**，但严格区分角色：

- **MCP 服务器**：独立进程，负责暴露三种能力：
  - **Tools**：可被模型调用的函数，带 JSON Schema 参数描述。
  - **Resources**：可读取的外部数据（文件、数据库记录、API 响应等）。
  - **Prompts**：预定义的提示模板，方便用户快速触发特定任务。
- **MCP 客户端**：运行在 Agent 宿主（如 OpenClaw 进程内），负责与服务器通信、管理连接生命周期，并把服务器能力翻译为模型能理解的 function declarations。
- **传输层**：目前主流有两种——
  - **Stdio**：通过子进程的标准输入/输出通信，适合本地工具。
  - **Streamable HTTP**：基于 HTTP 的 SSE（Server-Sent Events）通道，适合远程或云端部署的工具。

模型本身不直接与 MCP 服务器对话；它只与 OpenClaw 的 Agent 通信，Agent 再通过 MCP 客户端去真正的工具进程里执行调用。这个“中间层”精确地隔离了模型与工具的实现细节。

## 搭建一个可运行的 MCP 服务器

下面以 Python 为例，快速搭建一个“本地时间查询”的 MCP 服务器，并与 OpenClaw 对接。

**1. 安装 MCP SDK**

```bash
pip install mcp
```

**2. 编写服务器代码**

```python
# time_server.py
import asyncio
from datetime import datetime
from mcp.server import Server
from mcp.server.stdio import stdio_server

app = Server("time-server")

@app.tool()
async def get_current_time(location: str) -> str:
    """获取指定地点的当前时间（mock实现）"""
    # 实际应用中可接入时区API
    return f"{location} 当前时间: {datetime.now().isoformat()}"

async def main():
    async with stdio_server() as (read, write):
        await app.run(read, write)

if __name__ == "__main__":
    asyncio.run(main())
```

**3. 在 OpenClaw 中配置 MCP 连接**

假设你的 OpenClaw 实例支持 MCP 客户端（通过 YAML 配置或 UI 面板），添加如下配置：

```yaml
mcp_servers:
  - name: "time-server"
    transport: "stdio"
    command: ["python", "time_server.py"]
```

重启 Agent 会话后，`get_current_time` 会自动出现在模型的可用函数列表中。你可以直接让模型“帮我查一下上海的本地时间”，它会调用该函数并返回结果。

## 踩坑与排障要点

- **Stdio 与 SSE 的选择**  
  Stdio 简单可靠，但仅限于本地。如果你想把 MCP 服务器跑在服务器上供多个 Agent 共享，要改用 HTTP+SSE，并注意配置 CORS 和连接池。  
  报错 `Connection closed` 多半是因为客户端期望的传输协议与服务器实际暴露的不匹配。

- **工具参数 Schema 必须精确**  
  MCP 要求 `inputSchema` 是标准的 JSON Schema 格式。如果你偷懒用类型注解自动生成，要注意某些复杂嵌套或 `$ref` 可能会被模型误读。建议**显式声明 properties 和 required 字段**。

- **版本兼容问题**  
  MCP 规范仍在快速迭代。截至写作时，Python SDK 已更新至 0.9+，但 OpenClaw 的 MCP 客户端可能只兼容特定协议版本。如果遇到 `Unsupported protocol version` 错误，请检查 SDK 版本或降级到双方都支持的稳定版本。

- **工具返回值不要包含模型指令**  
  工具返回的文本会被直接放入模型上下文。切勿在里面夹带类似“请忽略上文”或“以 JSON 格式输出”的指令，否则可能污染模型决策。

- **安全隔离**  
  本地 Stdio 工具拥有当前用户的所有权限。如果要暴露文件写入、命令执行等危险操作，务必在工具内部加入白名单校验，或考虑用沙箱容器运行 MCP 服务器。

## 可复用的工程建议

1. **将高频工具封装成独立 MCP 服务**：文件系统操作、PDF 解析、SQL 查询、REST API 调用等，一次开发，所有支持 MCP 的 Agent 都能共用。
2. **充分利用社区 MCP 服务器**：GitHub 上已有大量现成的 MCP 服务器，覆盖谷歌搜索、浏览器操控、Notion、Obsidian 等。接入时优先选择维护活跃、有清晰 Tool 文档的项目。
3. **写好 Tool 注释，因为模型会读**：Tool 的函数名、docstring、参数描述都会直接影响模型是否会正确调用。用清晰、确定的语言，避免模糊描述。
4. **考虑可观测性**：让 MCP 服务器输出结构化日志（JSON 格式），方便接入你的 Agent 调试链路，快速定位调用失败原因。
5. **测试时模拟模型调用**：可使用 MCP Inspector（官方提供的调试工具）直接发送模拟请求，无需等 Agent 触发，能极大提升开发效率。

## 总结

MCP 并不是另一个“跨模型调用标准”的野心家，它解决的是一个更具体、更痛苦的问题：**为你驯服的模型插上外设，同时保持代码的整洁与长久可维护性**。对于 OpenClaw 用户，它意味着你终于可以把那些重复编写的工具回归为可组合的服务，Agent 的能力半径也不再受限于单一框架。

当协议标准化后，工具开发就变成了“写一个 MCP 服务器，所有 Agent 都受益”。这可能是 Agent 生态从手工小作坊走向规模化协作的第一步。

---

