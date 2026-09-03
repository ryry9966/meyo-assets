---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 35951
source: 综合讨论
publishedAt: 2026-09-03
---

## 背景

做 Agent 的人几乎都撞过同一堵墙：模型本身不缺推理能力，缺的是"够得着"的东西。想让它查数据库、读本地文件、调内部 API，每接一个工具就要写一套胶水代码，换个框架还得再写一遍。工具接到第三个以上，维护成本就开始失控。

2024 年 11 月 Anthropic 开源了 MCP（Model Context Protocol），把"模型如何调用外部工具"这件事标准化。目前主流客户端和 Agent 框架都已支持，OpenClaw 生态里也逐步把它当作插件的默认接入方式。

## 它到底解决什么问题

本质是连接层的标准化，一句话概括：把 M×N 变成 M+N。M 个客户端 × N 个工具，原来是 M×N 套集成代码；有了 MCP，工具方实现一次 server，客户端实现一次 host 逻辑，两头各自收敛。

MCP 基于 JSON-RPC 2.0，server 对外暴露三类原语：

- **Tools**：模型可主动调用的函数（查库、发请求、执行命令）
- **Resources**：可读取的上下文数据（文件、配置、文档片段）
- **Prompts**：预置的提示词模板

客户端启动后通过 `tools/list` 等方法运行时动态发现能力，而不是硬编码。这是它和传统 function calling 最大的区别：能力是协商出来的，插件可以热插拔。

## 最小上手路径

以 Python SDK 为例：

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("ops-tools")

@mcp.tool()
def query_order(order_id: str) -> str:
    """按订单号查询订单状态，返回 JSON 字符串。"""
    return db.lookup(order_id)

mcp.run()  # 默认 stdio 传输
```

在宿主（Claude Desktop 或你自己的 Agent 主循环）里注册这个 server，模型就能看到 `query_order`。本地进程用 stdio，远程服务用 Streamable HTTP。写自定义 server 之前，先看看官方仓库里 filesystem、git、fetch 这些现成实现，很多需求不必重造轮子。

## 踩坑点

1. **工具描述就是 API 文档，也是提示词的一部分。** 描述含糊，模型要么不调用，要么传错参数。写 description 的认真程度不应低于函数签名。
2. **stdio server 别往 stdout 打日志。** stdout 是协议通道，一行多余的 print 就会让客户端解析崩溃，日志一律走 stderr。
3. **权限问题。** MCP server 默认以你的本地权限运行，接入第三方 server 等于给了一个可执行任意代码的插件，务必审代码、最小授权，写操作加确认。
4. **注入风险。** 工具返回值会进入模型上下文，恶意数据源可以夹带指令（tool poisoning），对不可信结果要做隔离过滤。
5. **协议版本演进很快。** HTTP+SSE 已被 Streamable HTTP 取代，鉴权方式也在调整，旧 client 配新 server 可能直接连不上，排障先查版本。

## 可复用建议

- 内部 API 收敛成一个团队级 MCP server，而不是每人各写一套脚本
- 工具粒度宁小勿大，返回结构化 JSON，错误信息写清原因，方便模型自我纠正
- 所有工具调用留审计日志，排查"模型为什么调错了"全靠它
- 工具数量超过一屏就考虑拆分分组，太多工具会稀释模型的选择准确率

## 总结

MCP 没有让模型变聪明，它解决的是一个工程问题：能力接入的标准化。对 OpenClaw 用户来说，价值在于插件生态从此有了统一接口——写一次 server，所有支持 MCP 的宿主都能复用。建议从跑通一个官方示例开始，再把自己最常用的一个内部工具迁过去，体感会比读十篇介绍更直接。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/34c0345b3dff2065.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/7b0ccdcfaf304f41.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/f732f532a9fc9779.png)

