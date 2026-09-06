---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 36392
source: 综合讨论
publishedAt: 2026-09-07
---

## 背景：瓶颈不在模型，在"接"

过去一年做 Agent 的同学多半有同感：模型能力已经不是第一瓶颈，瓶颈在工具接入。让 Agent 读文件、查数据库、发消息、调内部 API，每接一个都要写一套 glue code——定义 schema、处理鉴权、管理子进程、拼接返回。Function calling 解决了"模型怎么调用函数"，但没解决"函数从哪来、怎么被发现、怎么跨宿主复用"。

## 问题：典型的 M×N 困境

M 个 Agent 宿主 × N 个工具/数据源。你写了个好用的 GitHub 工具，接 A 框架适配一次，接 B 客户端再适配一次；反过来每个宿主要集成 N 个工具。工具生态被锁死在各自的私有插件格式里，重复劳动全堆在适配层。

## MCP 的答案：把 M×N 压成 M+N

MCP（Model Context Protocol，Anthropic 2024 年底开源）的做法很克制：工具方实现一次 MCP Server，宿主方实现一次 MCP Client，中间用统一的 JSON-RPC 通信。核心抽象只有三个：

- **Tools**：模型可主动调用的动作（写文件、发请求）；
- **Resources**：宿主可控的上下文数据（文档、记录）；
- **Prompts**：用户触发的模板。

传输层主流两种：本地 stdio（子进程），远程 Streamable HTTP。注意：早期教程里的 SSE 传输已废弃，照抄旧文会踩坑。

## 最小可用：三步接入

Python 官方 SDK 十几行就能起一个 Server：

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("ops-tools")

@mcp.tool()
def query_order(order_id: str) -> str:
    """按订单号查状态，返回 JSON。仅当 order_id 形如 SO-1234 时调用。"""
    return db.lookup(order_id)

mcp.run()  # 默认 stdio
```

1. **写 Server**：一个函数，加一句"写给模型看"的 docstring；
2. **宿主注册**：在 OpenClaw 等宿主的 MCP 配置里填 command 和参数；
3. **先验证再挂载**：用官方 MCP Inspector 跑一遍 `tools/list` 和一次真实调用，确认没问题再接进 Agent。

## 踩坑点

- **stdout 是协议通道**：stdio 模式下往 stdout print 任何日志都会污染 JSON-RPC 流，日志一律走 stderr。
- **工具描述就是接口文档**：模型只看 name + description + schema 决定是否调用。描述要写清"什么时候该用、什么时候不该用"，含糊的描述等于误调用率飙升。
- **工具不是越多越好**：几十个工具的 schema 会实打实吃掉上下文窗口，按场景裁剪注入。
- **长任务要有反馈**：耗时操作用进度通知，否则宿主侧容易超时；工具尽量设计成幂等——模型重试是常态。
- **安全别裸奔**：第三方 Server 等于让渡执行权，返回内容里可能藏 prompt injection；文件类工具务必收窄目录范围，删除/支付类操作留人工确认。

## 可复用建议

- **先 stdio 后 HTTP**：本地先跑通，确有远程/多用户需求再上 Streamable HTTP。
- **粗粒度、少而精**：一个"创建工单"胜过五个底层 API 自由拼装；返回结果做精简和结构化，别把整张表塞回上下文。
- **把 MCP Server 当防腐层**：包装内部系统，而不是把生产库直接暴露给模型。
- **把 Inspector 和单测纳入流程**：协议层问题在接入 Agent 之前就该拦住。

## 总结

MCP 不会让模型变聪明，它做的是把"接入"这件脏活标准化：一次实现、处处复用，把工具生态从各框架的私有格式里解放出来。对个人用户，它是给 Agent 装手脚的统一插座面板；对团队，它值得纳入基础设施规划。建议从官方 SDK 模板起步，先跑通一个最小工具，再逐步替换掉你现有脚本里那些"只有你能维护"的 glue code。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/f07a4d2a4b9ddc43.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/71f8cd682ae7be98.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/34edd5579cc58af4.png)

