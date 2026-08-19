---
title: MCP 协议入门：别再把每个工具都写一遍胶水代码
feedId: 33889
source: 综合讨论
publishedAt: 2026-08-20
---

## 背景：Agent 工具集成的“N×M”困境

在 OpenClaw 这类 Agent 项目里，我们经常需要让模型访问外部能力：文件系统、GitHub、数据库、浏览器、内部 API。最原始的做法是给每个工具写一个 adapter，再在 prompt 里描述工具格式、在代码里处理调用与返回。工具越来越多后，问题开始暴露：

- 每接入一个新工具，就要写一套新的调用逻辑和返回解析；
- 换一个 Agent 宿主或模型，工具描述和调用格式可能要重新适配；
- prompt 里的工具说明和实际执行代码容易漂移，LLM 经常“乱调”参数；
- 权限控制、错误处理、进程管理各写各的，工程质量参差不齐。

这本质上是 **N 个 Agent 宿主 × M 个外部工具** 的集成爆炸问题。MCP（Model Context Protocol）想解决的就是这件事：用一个标准协议把“模型如何发现、调用、获取外部上下文”固定下来，类似 LSP 之于 IDE 插件生态。

## MCP 到底解决了什么问题

MCP 的核心不是“提供工具”，而是**统一上下文供给的接口**。它定义了三个角色：

- **Host**：承载模型的宿主，比如 OpenClaw、Claude Desktop、自研 Agent；
- **Client**：Host 内部与 Server 建立连接的客户端，负责协议通信；
- **Server**：暴露资源（resources）、工具（tools）和提示词（prompts）的独立进程。

有了这个分层，外部工具只需要实现一次 MCP Server，任何支持 MCP 的 Host 都能直接挂载，不再需要为每个宿主重写胶水代码。工具描述、参数 schema、返回结构都由 Server 自己声明，模型通过协议发现能力，而不是靠 prompt 里手写 JSON。

## 做法：从跑通到最小实现

**第一步：先跑一个现成 Server，理解链路。**  
以文件系统 Server 为例：

```bash
npx -y @modelcontextprotocol/server-filesystem /tmp/mcp-demo
```

这个命令会启动一个 MCP Server，暴露读文件、写文件、列目录等工具。然后在 OpenClaw 的配置里把该 Server 声明为 MCP 服务（通常在 `mcpServers` 配置块下），重启或热加载后，宿主内就可以通过 MCP 客户端调用这些工具。

**第二步：用 Python 写一个最小 MCP Server。**  
推荐使用官方 `mcp` SDK 的 `FastMCP` 封装，代码非常少：

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("demo")

@mcp.tool()
def add(a: int, b: int) -> int:
    """Add two integers and return the sum."""
    return a + b

if __name__ == "__main__":
    mcp.run()
```

保存为 `server.py` 后，用标准输入输出模式运行：

```bash
python server.py
```

然后用 MCP Inspector 或支持 MCP 的客户端连接测试。关键点在于：**工具函数的 docstring 就是模型看到的工具描述**，参数类型注解会被转换为 JSON Schema。描述写得越准确，模型调用越稳定。

**第三步：接入宿主并验证。**  
在 OpenClaw 或任意 MCP 客户端中配置该 Server 的启动命令和传输方式，然后让模型尝试执行 `add(2, 3)`。如果能返回 5，说明整条链路已经打通。

## 踩坑点

1. **stdio 和 SSE 混用**  
   本地进程通常用 stdio 通信，远程服务才用 SSE/HTTP。配置时如果传输类型写错，客户端会一直连不上，但错误日志往往不明显。

2. **Server 日志污染 stdio**  
   使用 stdio 模式时，Server 的标准输出必须只包含 JSON-RPC 消息。如果你在代码里 `print("server started")`，会破坏协议解析。日志一律写到 stderr。

3. **初始化握手失败被静默**  
   MCP 客户端连接时有一个 initialize 握手过程。很多客户端在握手失败后不会立刻报错，表现为“工具列表为空”。排查时优先检查 Server 是否能正常启动、协议版本是否匹配。

4. **工具描述太模糊**  
   如果 docstring 只写“处理数据”，LLM 会自行脑补参数含义，导致调用错误。描述里要写清楚用途、参数约束、返回格式。

5. **权限给得太大**  
   文件系统 Server 如果挂载整个用户目录，模型一旦误调用可能删改重要文件。始终只暴露最小必要目录或只读权限。

## 可复用建议

- **优先复用社区 Server**：GitHub、Filesystem、Postgres、Puppeteer 等已有官方或社区实现，不要自己造轮子。
- **把 MCP Server 当普通进程管理**：用 systemd、pm2 或容器方式托管，保证崩溃后能自动拉起。
- **用 Inspector 调试**：`npx @modelcontextprotocol/inspector` 可以可视化查看工具列表、调用和返回，比盲测高效得多。
- **参数 schema 要具体**：能用枚举就不用字符串，能限制范围就限制，减少模型乱调的概率。
- **统一命名和版本**：Server 名称保持稳定，升级时注意兼容性，避免宿主配置频繁变更。

## 总结

MCP 解决的不是“让模型变强”，而是**把 Agent 接入外部能力的工程复杂度降下来**。它让工具变成可插拔的标准化组件，让 OpenClaw 这类 Host 不再需要为每个工具单独写胶水代码。对于正在做插件化、自动化实践的团队来说，MCP 是一个值得投入的协议层——但前提是理解它的边界：协议统一了通信和描述，工具本身的质量和权限控制仍需要工程兜底。

---

