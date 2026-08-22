---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 34270
source: 综合讨论
publishedAt: 2026-08-23
---

MCP 在 Agent 圈已经变成高频词，但它的价值常常被说成“让模型连接一切”。如果只停留在这个层面，很容易把 MCP 当成又一个万能胶。实际回到工程场景，MCP 解决的是一个非常具体的问题：当 Agent 需要调用外部工具、读取外部数据时，我们能不能少写一些一次性的适配代码。

## 一、背景：工具接入的碎片化

在 MCP 之前，给 Agent 接能力通常有三条路：直接用 LLM 的 function calling 定义函数；用框架插件系统封装 API；或者自己写胶水代码暴露 HTTP 接口。问题不是做不出来，而是每接入一个系统，就要重复处理工具描述、参数 schema、返回值序列化、权限控制和错误传播。OpenClaw 这类自动化框架如果同时接文件系统、浏览器、数据库、Git，维护成本会随集成数量线性甚至指数上升。

## 二、MCP 真正解决的问题

MCP 把这些接入统一成一种客户端-服务器模式。它不提升模型智力，也不替代 RAG，它解决的是“接口碎片化”。具体是三件事：

1. **工具发现与描述一致**：Server 用结构化 schema 声明有哪些 tools、参数是什么、返回什么类型，客户端不用猜。
2. **客户端与工具解耦**：Agent 只需要实现一个 MCP Client，就能消费不同 Server 提供的能力，类似 IDE 与语言服务器的关系。
3. **资源与工具分离**：大文本、文件内容、日志等可以通过 resources 暴露，避免把所有东西塞进 tool 返回值。

## 三、最小可复现步骤

先安装 Python SDK：

```bash
pip install mcp
```

创建一个最小 MCP Server：

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("demo")

@mcp.tool()
def get_current_time() -> str:
    """Return current ISO 8601 time."""
    from datetime import datetime, timezone
    return datetime.now(timezone.utc).isoformat()

if __name__ == "__main__":
    mcp.run()
```

本地启动：

```bash
python server.py
```

如果你的客户端支持 mcp.json，可以这样注册 stdio 连接：

```json
{
  "mcpServers": {
    "demo": {
      "command": "python",
      "args": ["server.py"],
      "cwd": "/path/to/project"
    }
  }
}
```

然后 Agent 就能发现并调用 `get_current_time`。这个例子虽小，但已经覆盖 MCP 的核心链路：Server 声明工具、Client 连接、模型根据 schema 生成调用。

## 四、踩坑点

1. **stdio 与 SSE 混用**：本地 Agent 优先用 stdio；远程共享才考虑 SSE/HTTP。不要把 SSE 当默认，徒增部署复杂度。
2. **工具描述不精确**：docstring 和参数注释会直接进入模型上下文。模糊的描述会让模型不调用，或传错参数。建议每个工具写清楚“做什么、输入什么、输出什么、什么时候不该用”。
3. **返回值不可序列化**：MCP 工具返回必须是 JSON 可序列化内容。返回 datetime、自定义对象会导致连接报错。
4. **长时间任务放在 tool 里**：模型调用工具通常有超时。长任务应该返回任务 ID，另走轮询或事件通知。
5. **环境不一致**：Python 解释器、依赖路径、工作目录都会导致客户端拉起 Server 失败。优先使用 `uvx` 或绝对路径，并在客户端配置中显式指定 cwd。

## 五、可复用建议

- **先只读后写入**：第一批 MCP 化优先做查询、列表、搜索等只读操作，风险可控。
- **工具粒度做小**：一个工具一个职责，命名用“动词+名词”，如 `get_note_list`、`search_files`。
- **错误用结构化返回**：不要只抛异常，返回 `{"ok": false, "error": "..."}`，避免整个连接中断。
- **大内容用 resources**：文件全文、日志等走 resources，而不是塞进 tool 返回。
- **用 MCP Inspector 调试**：官方提供的调试器可以单独调 Server，验证 schema 和返回，再接入 Agent。

## 总结

MCP 解决的是工具接入的标准化问题，不是 Agent 的万能外挂。在 OpenClaw 实践中，最值得投入的是把高频、稳定、只读的操作先 MCP 化，获得可维护的接入层。其余复杂业务逻辑，仍然要留在你自己的服务里。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/56ff550b16dc4a79.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/5234fe0a0d60a98a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/f1f021f2904d73a6.png)

