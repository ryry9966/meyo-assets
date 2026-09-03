---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 35944
source: 综合讨论
publishedAt: 2026-09-03
---

## 背景

做 Agent 的人这两年几乎都被同一件事磨过：工具接入。模型侧的 function calling 早已可用，但每套 Agent 框架都有自己的工具描述格式、注册方式和调用约定。假设你有 M 个 Agent 应用、N 个内部系统（知识库、工单、监控……），理论上就要维护 M×N 份胶水代码，格式互不相通，改一处动全身。

MCP（Model Context Protocol）是 Anthropic 在 2024 年底开源的协议，目标就是把 M×N 压成 M+N：工具提供方把能力封装成 MCP Server，实现一次，任何支持 MCP 的宿主（Host/Client）都能自动发现并调用。类似 USB-C——不是更快，而是统一了接口。

## 它具体解决什么

MCP 基于 JSON-RPC 2.0，把"模型能用什么"标准化成三类原语：

- **Tools**：由模型决定何时调用的动作，比如查数据库、发消息；
- **Resources**：由应用侧控制的上下文数据，比如文件内容、API 返回；
- **Prompts**：用户侧可主动选择的模板。

传输层上，本地子进程走 stdio，远程服务走 Streamable HTTP。对做插件和自动化的同学来说，关键是它把工具的**定义、发现、鉴权、调用**从各框架的私有约定上升为协议层的事，工具生态第一次有了"通用件"的可能。

## 动手步骤

1. 用官方 SDK（Python / TypeScript）写一个最小 Server，先只暴露一两个工具，比如查内部 wiki；
2. 认真写工具描述和参数 schema——它们会原样进入模型上下文，是模型决策的唯一依据；
3. 在宿主的 `mcpServers` 配置里注册：`command`、`args`、`env`；
4. 先用 MCP Inspector 单独调试 `list_tools` / `call_tool`，确认入参出参符合预期，再接 Agent 流程；
5. 最后在 OpenClaw 这类宿主里实际跑一轮任务，观察模型是否在正确的时机选对了工具。

## 踩坑点

- **stdio 模式下 stdout 是神圣的**：任何 `print` 调试输出都会污染 JSON-RPC 流导致连接异常，日志必须走 stderr；
- 工具描述写得含糊，模型就会选错工具或瞎填参数。描述要当 prompt 来打磨和迭代，不是随手一行注释；
- 一次挂太多 Server/工具，上下文膨胀且选择准确率明显下降，按任务启用，不用的关掉；
- 协议版本在演进（HTTP+SSE 传输已被 Streamable HTTP 取代），客户端和服务端版本要对齐，遇到握手失败先查版本；
- 安全别省：工具描述本身可能被注入恶意指令，高危操作保留人工确认，文件类工具做目录白名单，密钥走环境变量而非仓库里的配置文件。

## 可复用建议

- **一个 Server 只管一个领域**，工具保持小而正交，别造万能大工具；
- 内部被反复封装过的 API（工单、监控、知识库）最值得沉淀成 MCP Server，一次投入，多个 Agent 端复用；
- 远程 Server 的鉴权放网关层统一处理，Server 本身保持无状态；
- 把 Inspector 调试当成接 Agent 前的固定工序，能省掉大量"模型为什么不调这个工具"的猜测。

## 总结

MCP 不会让模型变聪明，它解决的是工程问题：把工具生态从点对点的私有集成，变成可组合的标准件。对 Agent 和插件开发者，它的价值在于降低接入成本、提升复用性；理解了这一点，再评估哪些系统值得包成 MCP Server，投入产出比会清晰很多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/f307ae32299edf33.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/dbf145839d7588b1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/94841e17e12dd47c.png)

