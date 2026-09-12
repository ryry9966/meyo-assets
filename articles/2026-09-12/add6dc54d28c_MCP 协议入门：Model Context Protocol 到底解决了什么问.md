---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 37191
source: 综合讨论
publishedAt: 2026-09-12
---

# 背景

做 Agent 应用一年多，最重复的劳动其实不是写 prompt，而是写胶水代码：让 Agent 查数据库、调内部 API、抓网页、操作文件系统。每个宿主——IDE 插件、CLI 工具、聊天客户端——都要各自实现一遍，且互相不通。2024 年底 Anthropic 放出 MCP（Model Context Protocol），定位是"AI 应用的 USB-C 接口"，现在主流 Agent 框架和编辑器都已原生支持。

# 它到底解决了什么

核心是 **M×N 集成问题**：M 个模型应用 × N 个数据源/工具，传统做法要写 M×N 个适配器。MCP 把它压成 M+N——应用只实现一次 MCP Client，工具方只实现一次 MCP Server，中间靠标准协议通信。

另一个被低估的点：function calling 是各家模型的私有协议，MCP 是应用层协议，与具体模型解耦。换底座模型不用重写工具层，这对长期维护很关键。

概念上只有三个角色：

- **Host**：Agent / 客户端本体
- **Client**：Host 内管理一条与 Server 的连接
- **Server**：对外暴露能力

Server 暴露三类原语：`tools`（可执行动作）、`resources`（可读数据）、`prompts`（预置模板）。底层是 JSON-RPC 2.0，传输层常用 stdio（本地子进程）和 Streamable HTTP（远程服务）。

# 动手：最小可用的 Server

以 Python SDK 为例，十几行就能跑：

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("demo-ops")

@mcp.tool()
def get_order(order_id: str) -> dict:
    """按订单号查询订单，返回状态、金额与物流信息。"""
    return db.query_order(order_id)
```

把它写进客户端配置、重启会话，模型即可自主发现并调用。调试用官方 MCP Inspector，比盯日志快得多。

# 踩坑点

1. **工具描述就是 prompt。** 模型选不选某个工具，几乎完全取决于 description 清不清楚，参数用 enum 约束比事后校验有效得多。
2. **stdio 环境问题。** 本地 Server 是子进程，PATH、虚拟环境、Windows 编码都会导致"连接失败"。先在终端手动跑一遍启动命令再排查配置。
3. **同步阻塞。** Server 里写同步 HTTP 调用会卡住整个会话的消息循环，耗时操作务必 async 或丢后台任务。
4. **权限与注入。** 工具参数由模型填充，本质是不可信输入。文件路径、shell 命令做白名单；写操作默认要求人工确认，不要无脑 auto-approve。
5. **协议版本。** 早期教程里的 HTTP+SSE 传输已被 Streamable HTTP 取代，照抄旧文容易连不上。

# 可复用建议

- 一个 Server 只做一件事，工具粒度小而边界清晰；30 个清晰的小工具好过 10 个万能大工具。
- 把工具列表当 prompt 资产维护，纳入 code review。
- 每个 Server 配一个 Inspector 冒烟脚本，升级 SDK 后先跑一遍再上线。
- 锁定 SDK 版本，MCP 规范仍在快速迭代，语义变化不罕见。

# 总结

MCP 没有引入新魔法，它只是把集成接口标准化了。但工程上标准化的价值经常被低估：一次实现、处处接入，工具生态从私有脚本走向可共享的组件。如果你的 Agent 正在维护第三份重复的"查数据库"代码，就值得迁过来了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/68585d3218b113bc.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/002271821fa280fa.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/759bacef08cf7e30.png)

