---
title: MCP 协议入门：把 M×N 的集成问题变成 M+N
feedId: 35775
source: 综合讨论
publishedAt: 2026-09-02
---

## 背景

做 Agent 的人大多卡过同一个地方：模型能力够了，接工具很累。你想让 agent 查内部数据库、发工单、读日志，每接一个能力就要在框架里写一层适配——定义参数、写描述、对齐返回格式。换个客户端或运行时，这套胶水代码基本重写。工具侧同样如此：给不同 agent 平台做集成，得各写一份。M 个客户端 × N 个工具，就是 M×N 份适配。

## MCP 到底解决了什么

MCP（Model Context Protocol）2024 年底开源，核心主张一句话：把「模型怎么发现和调用工具」标准化。M×N 从此压成 M+N——工具方实现一次 MCP Server，任何实现 MCP Client 的 agent 都能用，反之亦然。它具体标准化了三件事：

1. **发现**（`tools/list`）：客户端启动时拉取工具清单和参数 schema；
2. **调用**（`tools/call`）：统一的 JSON-RPC 2.0 调用与错误格式；
3. **上下文供给**（resources / prompts）：除工具外还能暴露只读数据源和预设提示词。

传输层常见两种：stdio（客户端拉起子进程，适合本地）和 Streamable HTTP（远程服务）。要想清楚的是：MCP 不解决「模型选对工具」的问题，它只保证接得上；选得准不准，仍取决于描述质量和 agent 侧工程。

## 动手：最小可用示例

用 Python 的 FastMCP 写一个查内部文档的工具：

```python
from fastmcp import FastMCP

mcp = FastMCP("doc-search")

@mcp.tool()
def search_docs(keyword: str, limit: int = 5) -> str:
    """按关键词搜索内部文档，覆盖接口约定与项目细节。"""
    return do_search(keyword, limit)[:2000]

if __name__ == "__main__":
    mcp.run()
```

在客户端配置里以 stdio 方式注册：

```json
{
  "mcpServers": {
    "doc-search": {
      "command": "uv",
      "args": ["run", "server.py"],
      "env": { "DOC_INDEX_URL": "..." }
    }
  }
}
```

上线前用 MCP Inspector 或直接发 JSON-RPC 请求验证 list / call 两个动作，不要只靠 agent 里的表现反推。如果你的 agent 框架已有自己的插件体系，可以把 MCP Server 当作外部工具提供方，和现有插件并存，不必二选一。

## 踩坑点

- **描述写得含糊**。模型靠工具名 + 描述做选择，「search」不如「按关键词搜索内部文档，仅覆盖 2024 年后的内容」。
- **一次挂太多 server**。几十个工具全进上下文，token 膨胀，选择准确率明显下滑，按需启用。
- **stdio 部署后失联**。本地能跑，容器里命令路径、Python 环境、env 对不上就静默失败。先在目标环境手动执行一次 command 确认。
- **schema 过度复杂**。深层嵌套的参数结构小模型经常填错，尽量扁平、必填项少。
- **第三方 server 默认可信**。它是拿你的权限跑的进程，返回内容也可能夹带注入指令。先读代码，优先只读能力。

## 可复用建议

- 一个 server 只管一个领域，单 server 工具数控制在个位数；
- 把 description 当 prompt 写，明确「什么时候该用、不该用」；
- agent 侧记录每次 `tools/call` 的参数、耗时、错误，这是排障第一现场；
- 新工具先只读后写入，写入类操作加人工确认；
- 定期清理不再使用的 server 注册。

## 总结

MCP 的价值不是「让 agent 更聪明」，而是把工具接入从一次性胶水代码变成可复用的标准件。对同时维护多个 agent、多个工具的团队，这是实打实的维护成本下降。它没替你解决的部分——描述质量、权限控制、结果校验——依然是 agent 工程里要自己做好的功课。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/7144a3602a515287.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/321197926ce6fa6b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/65212bc52642a8b7.png)

