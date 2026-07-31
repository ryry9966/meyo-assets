---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 31070
source: 综合讨论
publishedAt: 2026-07-31
---

最近在多个 LLM 应用框架的更新公告里频繁看到 MCP（Model Context Protocol），Anthropic 在 2024 年底推出它之后，似乎每个做 Agent、做插件生态的团队都在讨论。上个月我在 OpenClaw 上整合外部 API 时踩了不少坑，回过头看，MCP 试图解决的正是那种“每个工具都得单独适配，每换一个模型就要重写胶水层”的碎片化问题。这篇从工程角度梳理一下：MCP 到底补了什么位，怎么用起来，以及早期实践里值得注意的点。

## 背景：工具调用与上下文注入的碎片化

假设要让一个 LLM 读取本地文件、查询数据库、调用 Slack 发消息。目前常见做法是：在代码里定义一组 function/tool schema，监听模型输出中的 function_call，再执行对应的业务逻辑，把结果以 system 或 user 消息形式拼回对话上下文。每个模型、每个框架（OpenAI、Anthropic、LlamaIndex、LangChain 等）对工具定义的格式、上下文注入方式、错误回传格式都有各自约定。换个模型，就得重写适配层；同一套工具要在多个 Agent 间复用，基本靠复制粘贴。

更麻烦的是，工具发现（Discovery）和动态注册基本没有统一标准。大部分实现都是硬编码一个工具列表，运行时无法增减，也不支持多服务间的工具编排。当你维护 5 个不同用途的 Agent 时，工具管理就变成一场灾难。

## MCP 解决了什么

MCP 的定位类似于 LSP（Language Server Protocol）之于编辑器生态，只不过服务对象换成了语言模型。它定义了一套标准化的客户端-服务端协议，基于 JSON-RPC 2.0，让模型（通过 MCP 客户端）能够：

- 发现远程提供的工具（`tools/list`）
- 读取资源（如文件、数据库记录）和资源模板（`resources/list`, `resources/read`）
- 发起工具调用（`tools/call`）
- 获得结构化的结果或错误

这意味着，一旦你实现了一个 MCP 服务端（比如一个 “文件系统服务器”），任何支持 MCP 的客户端（Claude Desktop、OpenClaw 自定义管道、未来的各种 Agent 框架）都可以即插即用地调用它，不再需要为每个模型单独写工具适配。协议里还内建了资源（Resources）和提示模板（Prompts）的概念，把“上下文注入”也标准化了：模型不但能调用工具，还能通过资源 URI 拉取上下文（如 `file:///project/README.md`），而不必在 prompt 里手工拼接大段文本。

**核心价值一句话：MCP 把“模型能调用什么”和“模型能看到什么”从专用胶水代码变成可复用、可发现、可组合的开放标准。**

## 动手搭建一个 MCP 服务端（Python）

下面用 `mcp` Python SDK（目前 Anthropic 官方提供）快速做一个服务端，暴露一个“获取天气”工具。

### 1. 安装依赖

```bash
pip install mcp
```

### 2. 编写服务端

```python
# weather_server.py
import asyncio
from mcp.server import Server, Tool, Resource, ResourceTemplate
from mcp.types import TextContent

app = Server("weather-server")

@app.tool()
async def get_weather(city: str) -> list[TextContent]:
    """获取给定城市的天气信息（模拟）"""
    # 在实际项目里替换为真实 API 调用
    weather_data = {
        "北京": "晴，22°C",
        "上海": "多云，28°C"
    }
    result = weather_data.get(city, "未找到该城市天气")
    return [TextContent(type="text", text=result)]

if __name__ == "__main__":
    # 使用 stdio 传输，方便与本地客户端对接
    from mcp.server.stdio import stdio_server
    asyncio.run(stdio_server(app))
```

### 3. 配置客户端（以 Claude Desktop 为例）

Claude Desktop 的配置文件 `claude_desktop_config.json`：

```json
{
  "mcpServers": {
    "weather": {
      "command": "python",
      "args": ["path/to/weather_server.py"]
    }
  }
}
```

重启 Claude Desktop 后，在对话里就可以直接问“北京天气怎么样”，模型会自动发现 `get_weather` 工具并发起调用。整个过程没有在 prompt 里写任何 function 定义，也没有拼接 JSON schema——全部由 MCP 协议在背后完成。

如果你在 OpenClaw 这类自定义代理管道中使用，也可以选择 HTTP+SSE 的传输方式，让 MCP 服务端作为一个独立进程运行，供多个客户端复用。

## 踩坑点与工程建议

实际集成时，以下几个点值得提前留意：

**1. 工具定义质量直接影响调用成功率**  
MCP 里的工具描述、参数说明会被直接转成模型的提示信息。如果描述模糊，模型就会胡乱传参。建议把 docstring 写得像 API 文档一样精确，标明参数类型、可选值、默认行为。

**2. 错误返回格式要严格遵守协议**  
早期测试时遇到一个坑：工具内部抛出未捕获异常，客户端收到的是 JSON-RPC 的内部错误，模型无法感知到业务语义。正确的做法是返回正常结果，但在内容里清晰说明业务错误（例如“城市非支持”），或者使用 MCP 定义的错误码，让客户端有机会重试或提示用户。

**3. 资源与工具的分工**  
不要把所有数据获取都做成工具。只读且可以被 URI 定位的数据适合用资源暴露，方便模型按需拉取，也方便做缓存与权限控制。工具应留给有副作用的操作或需要动态参数的计算。

**4. 多服务编排与命名空间**  
当有多个 MCP 服务器时，客户端可以通过配置文件统一管理，但工具名冲突会出问题。建议为每个服务端指定唯一的名称，并让工具名带上服务前缀（SDK 目前允许通过装饰器参数命名），避免覆盖。

**5. 安全模型仍需要自己兜底**  
协议本身不限制工具能做什么，一个暴露了 `rm -rf /` 的服务器会带来灾难。对于允许动态注册的服务端，务必做好权限控制和参数校验。当前社区推荐的做法是：把高危操作放在只读资源或需要人工确认的提示模板里，而不是全自动工具调用。

## 可复用的实践建议

- **先从已有社区 MCP 服务器入手**：GitHub 上已经有不少开箱即用的 MCP 实现（文件系统、数据库、搜索引擎、Jira 等），可作为参考或直接接入。
- **工具粒度适中**：一个“万能”工具会降低模型选择的准确性，而太多细碎工具又会增加上下文长度。遵循 Unix 哲学——一个工具做好一件事。
- **保持传输方式对客户端友好**：本地开发选 stdio，跨机器服务选 HTTP+SSE。如果想让 OpenClaw 这类浏览器端侧也能用，可以考虑通过轻量网关将 stdio 转成 WebSocket。
- **把工具配置基础设施化**：将 MCP 服务端列表、启动命令、环境变量纳入 IaC（如 Ansible、Docker Compose 或你的 Agent 部署脚本），避免人手一份配置导致不一致。

## 总结

MCP 补上了从“模型能力”到“外部世界连接”之间的标准层。它解决了工具调用的碎片化，把上下文注入提升为一等公民，让 Agent 开发中的“插件系统”第一次有了通用的语言。开始阶段协议和 SDK 都还在快速演进，部分细节（如 streaming、长连接下的会话管理）尚未完全稳定，但基本骨架已经足够拿来搭建一个企业内部的工具总线。

如果你正在用 OpenClaw 做自动化管道，或是折腾自己的 Agent 框架，强烈建议把工具暴露层按 MCP 规范来实现——哪怕现在只服务于一个模型，未来复用的收益也远超初期投入。

---

