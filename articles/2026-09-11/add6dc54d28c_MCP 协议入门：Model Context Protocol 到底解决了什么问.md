---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 36992
source: 综合讨论
publishedAt: 2026-09-11
---

## 背景：M×N 的集成困境

大模型应用真正落地，绕不开两件事：让模型调用工具、让模型读取外部数据。2024 年之前这件事没有统一标准——每个 Agent 框架有自己的插件格式，每个工具都得为每个宿主单独写一遍适配。假设有 M 个模型应用、N 个数据源，最坏要写 M×N 个连接器，社区重复劳动严重。

MCP（Model Context Protocol）是 Anthropic 在 2024 年底开源的协议，一句话概括：给「LLM 应用接入外部能力」定一个 USB-C 接口。宿主实现一次客户端，工具实现一次服务端，M×N 变成 M+N。

## 它具体解决了什么

MCP 定义了三个核心原语：

- **Tools**：模型可主动调用的动作（查库、发请求、写文件）
- **Resources**：可读取的上下文数据（文件内容、API 返回）
- **Prompts**：预置的提示词模板

加上统一的发现机制（服务端自描述能力列表）和两种传输方式（本地 stdio、远程 Streamable HTTP），宿主不再需要关心每个工具的私有协议。

## 上手步骤：写一个最小 MCP Server

以官方 Python SDK 为例，本地 stdio 模式四步走：

1. `pip install "mcp[cli]"`，用 `FastMCP` 起一个实例；
2. 用装饰器注册工具，参数类型注解和 docstring 就是给模型看的 schema；
3. 写业务函数，返回结构化结果，别返回巨型字符串；
4. 用 `mcp dev server.py` 打开 Inspector 调试，确认工具列表和调用链路正常，再接入宿主。

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("demo")

@mcp.tool()
def query_order(order_id: str) -> dict:
    """按订单号查询订单状态，返回状态与金额"""
    return db.get_order(order_id)

mcp.run()  # stdio 模式
```

## 踩坑点

- **stdout 污染**：stdio 模式下 stdout 只能跑协议消息，`print()` 调试日志会直接把会话打挂，日志一律走 stderr 或文件。
- **工具描述即路由依据**：模型选工具靠 name + description，描述含糊就会选错工具或编造参数。把描述当 API 文档写：什么时候该用、参数格式、返回什么。
- **工具数量膨胀**：一个 server 挂 40 个工具，模型选择准确率明显下降。按领域拆分，单个 server 控制在 10 个左右。
- **同步阻塞**：工具里塞了个 30 秒的同步请求，宿主侧直接超时。慢操作用 async，或返回任务 ID 供轮询。
- **SDK 版本错配**：协议在演进（旧版 SSE 已被 Streamable HTTP 取代），client 与 server 的 SDK 版本差太多会握手失败，先对齐版本再排查。

## 可复用建议

- 本地开发用 stdio，团队部署再切 Streamable HTTP + 鉴权；
- 先用 Inspector 调通再接宿主，不要在宿主里盲调；
- 返回值做截断和分页，防止大结果吃掉上下文窗口；
- 破坏性操作（删数据、对外发消息）强制加确认参数；
- 每个 server 单一职责，工具命名带域名前缀，方便模型区分。

## 总结

MCP 没有让模型变聪明，它做的是把集成成本从 M×N 压到 M+N：工具方写一次即可被所有兼容宿主使用，Agent 方接一次即可用上整个生态。对做自动化和插件的同学来说，与其为每个宿主写私有插件，不如把能力封装成标准 MCP Server——这才是它真正的工程价值。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/4060943f7e895d2a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/b669a31a11277cbc.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/62d17378ca3bd1ac.png)

