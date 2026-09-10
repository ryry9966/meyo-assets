---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 36994
source: 综合讨论
publishedAt: 2026-09-11
---

## 背景：从 M×N 到 M+N

在 MCP 出现之前，把模型接上真实世界是个体力活。一边是 M 个客户端（各种 Agent 运行时、IDE 插件），一边是 N 个数据源和工具（文件系统、数据库、CI、工单系统），想让它们互通，理论上要写 M×N 份胶水代码。每换一个客户端，工具层就得重写一遍。

MCP（Model Context Protocol）做的事很朴素：把"模型怎么调用工具、怎么读上下文"定义成一个开放协议。工具方实现一次 Server，任何支持 MCP 的客户端都能直接用，集成成本从 M×N 降到 M+N。

## 问题：它到底解决了什么

拆开看，MCP 标准化了三件事：

1. **能力发现**：客户端连上 Server 后可以列出有哪些 tools、resources、prompts，不需要预先硬编码。
2. **调用契约**：工具的入参出参用 JSON Schema 描述，调用走统一的 JSON-RPC 消息。
3. **传输方式**：本地进程用 stdio，远程服务用 Streamable HTTP，客户端不用关心 Server 是 Python 还是 Go 写的。

换句话说，MCP 没有让模型变聪明，它只是把"接线"这件事标准化了。plumbing 的价值在于：工具生态可以独立于任何单一 Agent 框架生长。

## 做法：四步接入

1. **明确角色**：OpenClaw 这类运行时是 Host，内部为每个 Server 建一条 Client 连接；你写的工具暴露为 Server。
2. **定义工具**：每个工具写清 name、description 和参数 schema。先从两三个只读工具开始，比如列目录、查状态。
3. **选传输**：本地调试用 stdio 起子进程；需要共享给团队时再部署成带鉴权的 HTTP 服务。
4. **用 Inspector 验证**：官方 MCP Inspector 可以脱离模型直接测工具调用，先把参数校验和返回格式跑通，再接进 Agent 链路。

## 踩坑点

- **description 是写给模型看的**。模型靠描述决定调不调、怎么调，"查询数据"这种模糊描述会导致误调用；要写清何时用、何时不用、参数含义。
- **工具太多会淹掉上下文**。几十个工具的 schema 本身就是 token 开销，模型选择准确率也会下降。按场景拆分 Server，或做工具过滤。
- **参数校验失败的错误要回传给模型**。直接抛异常终止对话是浪费，把"哪个字段错了、期望什么格式"作为错误信息返回，模型大多能自我修正。
- **stdio 模式吃环境**。子进程继承 shell 的 PATH 和编码，Windows 上的换行符、stderr 混入 stdout 都会让协议解析失败。日志一律走 stderr。
- **安全别外包给协议**。MCP 不解决鉴权细节和提示注入。第三方 Server 返回的 resource 内容要当作不可信输入；有写操作的工具默认加确认层。

## 可复用建议

- 工具粒度宁小勿大：小工具可组合，大工具的 schema 和错误分支都难维护。
- 每次工具调用记日志（参数 + 截断后的结果），排查"模型为什么这么调"时，这是唯一可靠的证据。
- 长耗时任务拆成"提交任务 + 查询进度"两个工具，别让一次调用挂住整个会话。

## 总结

MCP 本质上是一套接线标准，它解决的是集成复用问题，不承诺任何智能增益。真正的工程价值取决于你如何设计工具边界、描述质量和安全边界。建议从两三个只读工具起步，跑通 stdio 调试链路，再逐步沉淀自己的 Server 库——工具集是会随业务复利的资产，胶水代码不是。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/30dca91f28aa3318.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/4895ab0395b612f6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/0b0843576a2c6b2d.png)

