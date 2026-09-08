---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 36682
source: 综合讨论
publishedAt: 2026-09-09
---

## 背景

Agent 落地绕不开两件事：让模型拿到上下文，让模型能动手。前者是数据接入，后者是工具调用。在 MCP 出现之前，这两个问题没有统一约定——每个 Agent 框架对接每个数据源、每个工具，都要写一份定制胶水代码。N 个框架 × M 个工具，就是 N×M 份集成工作。

MCP（Model Context Protocol）把这组 N×M 收敛成 N+M：工具方实现一次 MCP Server，任意支持 MCP 的宿主都能用；宿主实现一次 MCP Client，任意 MCP Server 都能接入。

## 它到底定义了什么

MCP 本质是一套基于 JSON-RPC 2.0 的消息约定，核心抽象只有三个：

- **Tools**：模型可调用的动作，比如查数据库、发消息、跑脚本；
- **Resources**：可供读取的上下文数据，比如文件、配置、文档片段；
- **Prompts**：预置的提示模板。

传输层常用两种：stdio（本地子进程，简单可靠）和 Streamable HTTP（远程服务，支持会话）。整体是宿主 / Client / Server 三层：宿主是 Agent 应用，Client 负责与某个 Server 维持会话，Server 暴露能力。

## 最小落地路径

1. 明确要暴露什么能力，先只做一个 tool；
2. 用官方 SDK（Python 或 TypeScript）写 Server：声明 tool 的名称、描述、JSON Schema 入参，实现处理函数；
3. 本地调试：stdio 起进程，用 inspector 工具确认 `tools/list` 和 `tools/call` 正常返回；
4. 接入宿主：在配置里登记 Server 启动命令与参数，重启会话；
5. 观察日志：确认模型能稳定选中你的 tool，入参解析无误。

一个能被模型可靠调用的 tool，整条链路半小时内可以跑通。

## 踩坑点

- **描述就是接口。** 模型靠 name 和 description 选工具，写得含糊就会被误选或直接跳过。描述要写清"什么时候该用我"，而不是"我是什么"。
- **stdio 模式下，任何打印到 stdout 的日志都会污染协议流。** 日志一律走 stderr，这是新手最常见的翻车点。
- **返回值体积失控。** 把整张表、整个文件塞给模型，context 会被撑爆。在 Server 端做分页、截断和摘要。
- **入参校验要严格。** Schema 写得松，模型会"创造性"地填参数；返回结构化的错误信息，模型才有机会自我纠正。
- **安全别裸奔。** 第三方 MCP Server 以你的权限运行，工具返回内容还可能携带注入指令。敏感操作加人工确认，来源不明的 Server 不装。

## 可复用建议

- 一个 Server 只管一个领域，工具数量控制在个位数；粗粒度优于大量碎片工具。
- 按 API 设计的标准对待 tool 描述，它直接决定调用准确率。
- 工具尽量幂等、无状态；出错时返回模型能读懂的结构化信息，而不是裸异常。
- 优先 stdio 起步，确有远程共享需求再迁到 Streamable HTTP。

## 总结

MCP 解决的是连接层的标准化问题，不是智能化问题。它不会让 Agent 更聪明，但把"接一个新工具"的成本从按天降到按小时。对 OpenClaw 这类插件生态来说，价值很直接：能力提供方写一次，所有宿主可用。协议本身不复杂，难点其实在工具设计——描述、粒度、错误处理，这些老工程问题在 MCP 语境下被放大了。建议先跑通一个最小 tool，再谈架构。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/7cadedfe001cdfd2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/4382153fd2d33b12.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/1640d1dc7f0a9641.png)

