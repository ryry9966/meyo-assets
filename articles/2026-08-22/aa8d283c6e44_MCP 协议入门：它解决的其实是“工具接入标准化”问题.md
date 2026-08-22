---
title: MCP 协议入门：它解决的其实是“工具接入标准化”问题
feedId: 34144
source: 综合讨论
publishedAt: 2026-08-22
---

## 背景：一个老问题的新包装

在 OpenClaw 这类 Agent 项目里，接入外部能力通常有三类：本地脚本、HTTP API、第三方插件。每接一个工具都要写适配：参数怎么传、返回怎么格式化、错误怎么处理、上下文怎么塞给模型。工具多了以后，适配层变成重复劳动。

MCP（Model Context Protocol）做的事情并不神秘：它定义了一套统一的 client/server 协议，让工具、资源、提示词可以通过同一套接口暴露给 Agent。可以把它类比成“工具接入界的 LSP”：以前每个编辑器给每种语言写插件，现在语言服务只要实现 LSP，编辑器实现一次 client 就能用。

MCP 解决的不是模型能力问题，而是工程化接入问题。

## 问题：没有 MCP 之前，接入成本高在哪？

1. 每个工具的 schema 写法不一致：有的用 JSON Schema，有的用自然语言描述，导致模型调用不稳定。
2. 连接方式五花八门：stdio、HTTP、WebSocket、IPC，客户端要分别处理。
3. 上下文与工具分离：工具返回值要手动拼进 system prompt，格式容易脏。
4. 权限和生命周期管理散落：每个插件各管各的，出问题难排查。

这些问题的本质是缺乏一个统一协议。MCP 的价值就是把这些接口形状固定下来。

## 做法：最小化跑通一个 MCP Server

以 Python 为例，先跑通一个返回时间的工具。

1. 安装 SDK：

```bash
pip install mcp
```

2. 写 server，暴露一个 `get_current_time` 工具：

```python
from mcp.server.fastmcp import FastMCP
import datetime

mcp = FastMCP("time-server")

@mcp.tool()
def get_current_time(timezone: str = "UTC") -> str:
    """返回当前时间，IANA 时区可选，如 Asia/Shanghai。"""
    tz = timezone or "UTC"
    # 真实实现应处理时区转换，这里只做演示
    return f"{datetime.datetime.now():%Y-%m-%d %H:%M:%S} {tz}"

if __name__ == "__main__":
    mcp.run()
```

3. 在 OpenClaw 配置 MCP client 连接 stdio server：

```json
{
  "mcpServers": {
    "time": {
      "command": "python",
      "args": ["time_server.py"]
    }
  }
}
```

4. 验证：在 Agent 里传一句“现在上海几点”，如果工具描述清晰，模型会调用 `get_current_time`，返回文本被注入上下文。

远程部署时，可以用 SSE/HTTP transport，但需要额外处理鉴权。

## 踩坑点

1. **协议版本要锁定**。MCP 发展快，不同 SDK 版本可能不兼容。建议在依赖文件里写死版本，不要用 latest。
2. **工具描述比代码更重要**。模型靠 description 决定是否调用，太含糊会导致不调或乱调。描述里要写清楚参数含义、返回格式和副作用。
3. **返回内容要克制**。不要把整个数据库查询结果塞进 TextContent。只返回模型需要的最小字段，否则上下文很快爆掉。
4. **stdio 适合本地，远程用 SSE**。stdio 简单稳定，但不适合跨机器；SSE 方便但断线重连、鉴权要自己兜。
5. **MCP server 不要写重业务**。它应该是一个薄适配层，重逻辑放到后端服务，MCP 只做翻译和转发。否则每加一个工具都要改 server，最后又变成单体插件。
6. **权限要显式声明**。不要默认信任 MCP server 的所有工具。能只读就别暴露写操作，能限制参数就限制。

## 可复用建议

- 先从小工具试点，不要一上来把公司所有 API 都接进 MCP。
- 给每个工具固定输出结构：`{ ok: boolean, data: ..., error: ... }`，方便 Agent 判断。
- 记录 tool call 失败率。如果某个工具经常被错误调用，先优化 description，而不是改代码。
- 把 MCP server 当基础设施，统一配置在项目里，避免每个人本地各写一套。
- 使用环境变量管理密钥，不要把 API key 硬编码在 server 文件里。

## 总结

MCP 解决的是“工具接入标准化”问题，让一个 Agent 客户端可以复用多个 MCP server，而不是为每个工具写一套插件。它不会让你的 Agent 变聪明，但能显著降低接入成本和排查难度。对 OpenClaw 实践者来说，适合把它当作稳定的工具层协议，而不是追逐热点的概念。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/5f46bdb51c2e26c1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/b76685c36ed8f97a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/ad7c3e08ae804249.png)

