---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 36132
source: 综合讨论
publishedAt: 2026-09-05
---

## 背景：每个 Agent 都在重复造轮子

做 Agent 开发绕不开一个现实：模型本身只会生成文本，真正干活要靠外部工具——查数据库、调内部 API、操作文件系统。早期的做法是把每个工具的功能描述和参数 schema 直接塞进 system prompt，再自己写一套 function calling 的胶水代码。

M 个应用接 N 个工具，最坏情况是 M×N 套集成代码。各家模型的调用格式还有细微差别，换个模型，描述层就得重写一遍。

## MCP 解决了什么

Model Context Protocol（MCP）是 Anthropic 在 2024 年底开源的协议，思路类似 LSP（Language Server Protocol）：把「模型 ↔ 工具」这一层标准化，把 M×N 的问题收敛成 M+N。

核心架构是三个角色：

- **Host**：Agent 应用本体（如 OpenClaw、Claude Desktop）
- **Client**：Host 内部维持与单个 server 连接的组件
- **Server**：暴露能力的独立进程，可以是本地 stdio 进程，也可以是远程 HTTP 服务

Server 对外暴露三类原语：

- **Tools**：模型可主动调用的动作（发请求、写文件）
- **Resources**：可读取的上下文数据（文件、配置）
- **Prompts**：预置的提示词模板

协议底层是 JSON-RPC 2.0，传输层支持 stdio 和 Streamable HTTP。

## 最小实践：写一个 MCP Server

用 Python SDK 二十行内就能跑通：

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("demo-tools")

@mcp.tool()
def query_order(order_id: str) -> str:
    """按订单号查询订单状态，返回 JSON 字符串"""
    return db.lookup(order_id)

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

在 Host 的配置里注册这个 server，重启后模型就能看到 `query_order` 这个工具。调试推荐官方的 MCP Inspector（`npx @modelcontextprotocol/inspector`），不用反复重启 Host。

## 踩坑点

1. **stdio 模式下别往 stdout 打日志**。stdout 被协议帧占用，一句 `print("debug")` 就能让连接解析失败。日志走 stderr 或写文件。
2. **工具描述是写给模型看的**。`def get_data(id)` 这种没有用途说明的定义，模型要么不调用要么乱传参。docstring 要写清楚：什么时候用、参数格式、返回什么。
3. **工具数量失控**。挂到三十个工具后，模型选错率明显上升。按业务域拆分 server，或者在 Host 层做工具分组和子 Agent。
4. **权限意识**。MCP server 跑在你的本机权限下，一个能执行 shell 的 server 等于把机器交给模型。生产环境要做白名单和只读约束。
5. **Windows 下 stdio 的编码和启动方式**。子进程 shell 选择、UTF-8 配置，跨平台部署前提前验证。

## 可复用建议

- **先做能力盘点再动手**：高频、稳定、幂等的操作优先暴露；危险操作外面加确认层。
- **Resources 和 Tools 别混用**：只读数据用 resource，有副作用的用 tool，模型对二者的调用策略不同。
- **错误信息返回给模型而不是抛异常中断**——模型看到结构化报错，往往能自我修正参数后重试。
- **把 server 当独立服务管理**：有版本号、有日志、有健康检查，别把逻辑写死在 Host 里。

## 总结

MCP 的价值不在技术多先进，而在把一个混乱的集成问题收敛成了标准接口。对实践者来说，它意味着工具生态真正可复用：server 写一次，任何兼容 Host 都能接。建议从一个只读工具开始试，先跑通 stdio 链路，再考虑远程部署和权限治理。协议本身还在演进，但「标准化工具层」这个方向已经没有悬念。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/2658ea6542a8a437.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/2ed22202dcac1365.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/74764199de9f0065.png)

