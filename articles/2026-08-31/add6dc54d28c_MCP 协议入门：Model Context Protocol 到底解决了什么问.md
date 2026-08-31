---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 35563
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景：不是新模型，是新的接线方式

MCP 全称 Model Context Protocol，最早由 Anthropic 提出。它解决的不是“模型变强”的问题，而是 Agent 工具集成的工程问题：同一个知识库、同一个内部 API，如何只写一次，就能被 OpenClaw、其他 Agent Host 或 CLI 复用。

在没有 MCP 时，常见链路是：每个宿主都要为工具写 adapter，N 个工具 × M 个宿主，出现一堆胶水代码。OpenClaw 作为 Agent Host，最需要的不是把调用 JSON 包装得更好看，而是有一套统一的发现、调用、读取资源和提示词模板的协议。MCP 把这层约定固定下来。

## MCP 实际解决的三件事

1. **统一发现**：通过 `tools/list`、`resources/list`、`prompts/list`，宿主能知道一个 server 有什么能力，而不是写死函数表。
2. **统一调用**：基于 JSON-RPC 2.0，使用标准消息，不同语言实现可以互通。
3. **统一生命周期**：初始化、能力协商、心跳/关闭，让宿主和服务端不靠隐式约定。

角色上要分清：

- **Host**：OpenClaw 这类宿主程序；
- **Client**：Host 内到某一个 server 的连接；
- **Server**：实际提供 Tools、Resources、Prompts 的进程或服务。

其中 Tools 是动作，例如查询、创建；Resources 是数据，例如文件、知识库、工单；Prompts 是可复用提示模板。

## 做法：先跑一个最小的 stdio server

建议第一版不要上 HTTP，直接用 stdio。以下是一个最小 Python MCP server 骨架：

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("demo")

@mcp.tool()
def echo(text: str) -> str:
    """返回去除首尾空格后的文本；只读，无副作用。"""
    return text.strip()

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

在 OpenClaw 中注册的方式通常是声明一个 `mcpServers` 配置：

```json
{
  "mcpServers": {
    "demo": {
      "command": "python",
      "args": ["-m", "my_mcp_server"]
    }
  }
}
```

注册后先做 smoke test：确认 `tools/list` 能看到 `echo`，再让 Agent 用一句话调用。比如输入“用 echo 处理一下 `  hello  `”，模型应当能选择工具并完成调用。

## 踩坑点

- **stdout 必须留给协议**。stdio server 的所有日志要写 stderr，否则会污染 JSON-RPC，导致 host 解析失败。
- **描述就是选路信息**。不要把 tool 名称写成 `get` 或 `handle`。模型只能靠 name + description + schema 判断该不该调用、怎么调用。
- **不要混淆 Resources 和 Tools**。只读数据用 resource；会产生写操作的，必须做成 tool，并标记副作用。
- **不要过早在远程 HTTP 上做复杂鉴权**。先本地 stdio 验证能力，再切 Streamable HTTP，并加 TLS、token、最小权限。
- **大结果不要直接塞进 tool result**。返回一个 resource URI 或摘要，让宿主按需拉取。
- **写操作要可确认**。删除、发送、审批类工具应单独拆分，不要和查询混合在一个 tool 里。

## 可复用建议

- 命名用 `动词_名词`，例如 `search_customer`、`create_ticket`。
- 每个 tool 描述包含：用途、输入、输出、是否只读、失败时返回什么。
- 用 JSON Schema 约束参数，减少模型幻觉参数。
- 给资源设计稳定 URI，不要用临时路径作为 ID。
- 先用 `tools/list` 验证注册，再用真实 Agent 做最小闭环，最后才加复杂逻辑。

## 总结

MCP 不是 Agent 的“大脑升级”，而是把工具、数据和提示模板从宿主绑定中抽出来。它的核心价值是把 M×N 的集成成本降为 M+N。对 OpenClaw 用户来说，最重要的是先理解 Host/Client/Server 和 Tools/Resources/Prompts 的边界，然后从一个小而明确的 stdio server 开始。不要一上来追求大而全的平台化，先让一个工具可发现、可调用、可调试，再谈扩展。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/8c0d6f6131a11cd7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/381cb639d0ea4827.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/6b5cfc77f8e35920.png)

