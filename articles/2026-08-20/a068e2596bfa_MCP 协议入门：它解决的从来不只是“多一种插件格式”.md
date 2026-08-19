---
title: MCP 协议入门：它解决的从来不只是“多一种插件格式”
feedId: 33877
source: 综合讨论
publishedAt: 2026-08-20
---

# MCP 协议入门：它解决的从来不只是“多一种插件格式”

很多做 Agent 开发的团队第一次看到 MCP，第一反应是：这不就是又一种插件标准吗？实际上，如果你已经在维护 OpenClaw、自定义 Agent 或一堆自动化脚本，真正折磨人的问题要具体得多：每接一个工具就要写一套适配；工具返回格式五花八门；上下文散落在不同系统里；权限管控完全没有统一入口。

MCP 要解决的，不是“让插件更好写”，而是把 Agent 获取上下文和调用工具这条链路标准化。

## 背景：没有 MCP 之前，集成是一场 N×M 的消耗战

在 MCP 出现前，接入一个内部工具通常有几种做法：

- 直接用 function calling 写 JSON schema；
- 包一层 REST API，再让 Agent 通过 HTTP 调用；
- 使用各类插件系统，每个框架一套接口。

问题很明显：不同 Agent 框架、不同工具之间形成 N×M 组合爆炸。一个内部 API 想同时给 OpenClaw、Claude Desktop、自研 Agent 用，往往要写多次适配。更麻烦的是，工具描述、错误格式、上下文传递、权限审批这些“暗能力”完全依赖实现者个人习惯，换一个人维护就得重新理解一遍。

## MCP 到底解决了什么

MCP 的全称是 Model Context Protocol，核心是定义了一套统一的客户端-服务端协议：

- **Host/Client**：运行 Agent 的宿主程序，负责发现和调用 MCP server；
- **Server**：暴露工具、资源、提示词的独立进程；
- **传输层**：基于 JSON-RPC，支持 stdio、SSE、streamable HTTP；
- **能力抽象**：`tools` 表示可执行动作，`resources` 表示可读取的上下文，`prompts` 表示可复用的提示模板。

它不是“一种新插件格式”，而是把“工具如何被发现、调用、返回错误”这件事固定下来。一个 MCP server 写好后，可以在多个客户端里复用；权限审批、调用审计可以放在 client 侧统一做。对工程化团队来说，这才是真正的减负。

## 做法：先跑通一个最小只读 MCP server

下面以 Python 和 OpenClaw/Claude Desktop 兼容的配置为例，步骤尽量从简。

### 1. 定义暴露面

不要一上来写代码。先回答：这个 server 到底让 Agent 做什么？建议只读优先。比如：

- `search_incidents(keyword, limit)`：按关键字搜索工单；
- `get_doc_summary(path)`：读取文档摘要；
- `list_recent_deploys(service)`：列出最近部署。

能只读就不要写，能查表就不要执行命令。

### 2. 实现最小 server

用官方 Python SDK 写一个最小实现：

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("oncall-mcp")

@mcp.tool()
def search_incidents(keyword: str, limit: int = 10) -> list[dict]:
    """按关键字搜索最近工单，返回标题、状态、负责人。"""
    # 这里接你的内部 API 或数据库
    return [
        {"title": "支付超时", "status": "open", "owner": "zhang"}
    ]
```

工具描述和参数描述不是摆设，它们会直接影响模型是否选择这个工具、如何填参数。

### 3. 在客户端侧注册

OpenClaw 或 Claude Desktop 的配置思路类似，通常在配置文件里声明 `mcpServers`：

```json
{
  "mcpServers": {
    "oncall": {
      "command": "/abs/path/to/python",
      "args": ["/abs/path/to/server.py"],
      "env": {
        "INTERNAL_API_TOKEN": "xxx"
      }
    }
  }
}
```

注意 `command` 最好写绝对路径。本地用 stdio 就够了，不要一上来搞 SSE。

### 4. 验证

先用 MCP Inspector 跑通：

```bash
npx @modelcontextprotocol/inspector python server.py
```

在 Inspector 里确认 `tools/list` 能看到工具，手动调一次 `tools/call` 返回正常，再接入真实客户端。

## 踩坑点

这些是我在接入时遇到的真实问题，不是理论提醒。

- **stdio 启动失败**：客户端启动 MCP server 不继承完整 shell 环境。常见表现是 `python: command not found`。写绝对路径，或者显式传入 `env`。
- **工具描述太模糊**：模型不知道什么时候该用工具，就会乱调或不调。`search_incidents(keyword)` 这种描述如果只写“搜索”，实际调用质量会很差。
- **在 server 里做重业务**：MCP server 应该短响应、无状态。长时间任务、复杂鉴权、大量数据同步都不该放在一个 tool 调用里完成。
- **误以为 MCP 自身能管权限**：它不管。一个 MCP server 被注册后，能做什么完全取决于 client 是否做了审批、只读约束、调用审计。生产环境不要裸奔。
- **transport 选型过重**：本地单机工具用 stdio 足够稳定；只有多客户端远程共享时才考虑 SSE/streamable HTTP。过早引入远程传输只会增加排障复杂度。

## 可复用建议

如果有多个团队要写 MCP server，建议落地一版最小 checklist：

- **只读优先**：默认不给写权限，写操作单独拆 server 或加审批；
- **schema 完整**：每个参数的 `type`、`description`、`required` 都写清楚；
- **错误返回统一**：不要只在 stderr 打印，返回结构里给 `error` 字段；
- **日志走 stderr**：stdout 必须留给 JSON-RPC，否则协议会被污染；
- **控制工具数量**：单个 server 暴露 5–15 个工具为宜，按业务域拆分；
- **保留调用记录**：在 client 侧记录 `tools/call` 的参数和返回，方便排障。

## 总结

MCP 不是又一种插件流行趋势，它解决的是 Agent 与外部能力之间的标准化问题：减少重复集成、统一工具抽象、让权限和审计有地方落。

但协议本身不解决“工具设计是否合理”。对 OpenClaw 用户来说，更务实的路径是：先写一个只读查询类 MCP server，跑通工具发现、调用、错误返回和客户端审批，再考虑扩大暴露面。这比看十个 demo 有用得多。

---

