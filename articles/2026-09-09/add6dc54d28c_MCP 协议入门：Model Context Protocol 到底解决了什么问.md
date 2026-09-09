---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 36726
source: 综合讨论
publishedAt: 2026-09-09
---

## 背景

过去一年做大模型应用，最耗时的往往不是调提示词，而是"接线"：让模型能查数据库、调内部 API、操作文件系统。M 个应用接 N 个工具，就要写 M×N 套定制胶水代码，且互不通用。MCP（Model Context Protocol）是 Anthropic 在 2024 年底开源的协议，目标就是把这个 M×N 问题降成 M+N：工具方实现一次 MCP Server，应用方实现一次 MCP Client，中间走统一协议。

## 它到底解决什么

MCP 本质是一份基于 JSON-RPC 2.0 的接口约定，核心抽象只有三个：

- **Tools**：模型可调用的动作（有副作用，如发请求、写文件）
- **Resources**：应用可读取的数据（只读，如配置、文档）
- **Prompts**：预置的提示词模板

可以把它理解成"AI 应用的 USB-C"：以前每个设备一种线，现在统一插口。对做 Agent 和插件体系的同学，价值在于生态复用——社区现成的 filesystem、git、数据库 Server 可以直接接进自己的框架，不用重复造轮子。

## 最小实践步骤

用 Python SDK 写一个最简单的工具 Server，大约十行：

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("demo")

@mcp.tool()
def get_weather(city: str) -> str:
    """查询指定城市的当前天气，city 传中文或拼音城市名。"""
    return f"{city}: 晴, 26°C"

mcp.run()  # 默认 stdio 传输
```

然后在宿主（Claude Desktop、自研 Agent 等）配置里声明启动命令即可。握手后宿主会拿到工具清单和参数 Schema，模型决定何时调用，宿主负责执行并回传结果。整个链路没有魔法，就是标准的"能力发现 → 模型决策 → 工具执行 → 结果回填"。

## 踩坑点

1. **stdio 传输下别往 stdout 打日志。** stdout 只能走协议消息，一行 `print` 调试输出就会破坏 JSON-RPC 帧，日志一律走 stderr。这是新手第一大坑。
2. **工具描述写得太随意。** 模型只看名称、描述和 Schema 做决策，描述含糊就会乱调用或拒调。把描述当成给模型看的 API 文档来写，说清参数格式和边界条件。
3. **教程版本过时。** 2025 年起的规范用 Streamable HTTP 替代了旧的 SSE 传输，大量旧文还在教 SSE，接不上先核对协议版本和传输方式。
4. **安全别裸奔。** 运行第三方 MCP Server 等于执行不可信代码；工具返回的内容也可能携带注入指令。生产环境要做来源审计、工具白名单和结果审查。
5. **工具数量失控。** 一次挂几十个工具会撑上下文、降低选择准确率，按场景拆分、按需挂载。

## 可复用建议

- 只读数据用 Resource，有副作用的动作用 Tool，别混用。
- 工具粒度要小，宁可多个原子工具，也不要一个万能大工具。
- 长耗时任务设计成返回任务 ID 的异步模式，避免阻塞调用链。
- 联调前先跑官方 Inspector，能完整看到握手和调用报文，排查效率高很多。

## 总结

MCP 不是智能增强，它解决的是纯工程问题：接口标准化与生态解耦。它的价值随接入方数量增长而放大——今天认真写的一份工具描述，明天就可能被整个生态复用。建议从现成 Server 跑通一条最小链路，再逐步替换成内部工具，比一上来就自研框架划算得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/d9c4f55fdc350b49.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/c5cb1a6c03910b3f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/fed6df43858a7606.png)

