---
title: MCP 协议入门：统一 Agent 上下文接入的工程标准
feedId: 34204
source: 综合讨论
publishedAt: 2026-08-22
---

## 背景：Agent 集成的 N×M 困境

过去要让一个 Agent 读本地文件、查数据库、调浏览器、发消息，通常需要为每个能力写适配代码。一个 Agent 项目接 5 个工具，两个 Agent 项目就是 10 段胶水；如果工具提供方升级接口，所有 Agent 都要跟着改。MCP（Model Context Protocol）把这种集成从“每个 Host 连接每个工具”的单点定制，变成“Host 只实现一个 MCP Client，工具提供方只实现一个 MCP Server”。

协议基于 JSON-RPC 2.0，Server 可以暴露三类对象：

- **Resources**：资源，如文件、数据库记录；
- **Tools**：工具，模型可调用；
- **Prompts**：提示模板。

Client 通过 `initialize`、`tools/list`、`tools/call` 等标准方法通信。

## 问题：标准协议并不能替你设计工具

很多人觉得接入 MCP 就自动获得“工具调用能力”，其实不是。MCP 只解决传输和发现，不解决“这个工具是否值得被调用”。实践中真正耗时的是：工具输入输出 schema 设计、描述文案、错误返回格式、权限边界。

## 做法：从最小 MCP Server 开始

以 Python 官方 SDK 为例，写一个只暴露一个工具的最小 Server：

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("files")

@mcp.tool()
def count_lines(path: str) -> dict:
    """统计文本文件行数，path 必须是绝对路径。"""
    try:
        with open(path, encoding="utf-8") as f:
            return {"path": path, "lines": len(f.readlines())}
    except FileNotFoundError:
        return {"error": "file_not_found"}

if __name__ == "__main__":
    mcp.run()
```

保存为 `server.py`，用 `python server.py` 启动，默认通过 stdio 与 Host 通信。

接入 OpenClaw 或其他 Agent Host：不同 Host 的配置位置略有差异，OpenClaw 一般在 MCP 设置或配置目录的 JSON 中增加一项：

```json
{
  "mcpServers": {
    "files": {
      "command": "python",
      "args": ["/abs/path/to/server.py"]
    }
  }
}
```

重启 Host 后，在可用工具列表中应该能看到 `count_lines`。先不要急着让模型调用，建议先用 MCP Inspector 连接同一个 Server，手动调用一次确认返回 JSON 符合预期。

## 踩坑点

- **stdio 路径与环境**：配置文件里的 `command` 不会经过 shell，`~`、管道、环境变量展开都可能失效。用绝对路径，或把可执行文件放进 PATH。如果依赖虚拟环境，`command` 应指向虚拟环境里的解释器。
- **schema 即文档**：工具的 `inputSchema` 和 description 会直接进入 LLM 上下文。如果描述只写“查询文件”，模型在需要统计行数时可能不调用它。把用途、参数约束、返回结构、可能的副作用写清楚。
- **返回必须可序列化**：不要返回生成器、日期对象、文件句柄。MCP Server 内部异常如果不捕获，可能直接断开连接或返回无意义错误，导致 Host 侧工具状态异常。
- **传输方式差异**：本地开发用 stdio 简单可靠；部署到远程时如果用 Streamable HTTP，要处理鉴权、CORS、服务保活。不要把本地 stdio 的假设带到远程。
- **版本与生命周期**：协议仍在演进，Server 和 Client 的版本需要互相兼容。开发时经常出现 Host 退出后 Server 子进程残留，注意清理；有些 Host 在 Server 崩溃后不会自动重连，需要手动重启。

## 可复用建议

1. 用官方 SDK（Python/TS）写 Server，不要手写 JSON-RPC。
2. 一个 Server 只做一类事，比如一个文件 Server、一个浏览器 Server，避免权限过大。
3. 先脱离模型用 MCP Inspector 调通，再让 Agent 使用。
4. 生产 Server 增加超时、日志、只读权限开关，尤其是访问文件系统或 Web 的工具。
5. 记录每个工具的调用输入输出，便于排查模型误用。

## 总结

MCP 真正解决的问题，是把 Agent 外部上下文接入从项目级胶水代码，变成可复用、可发现、可替换的标准组件。它不替代工具设计，也不自动保证安全；但它让“接一个工具”这件事，不再因为换一个 Host 就重新做一遍。对 OpenClaw/Agent 实践者来说，早一点把自发工具改成 MCP Server，后面复用成本会明显下降。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/b697ef0e988222ca.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/0a7771a36bbe17e6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/a5e790be55c52690.png)

