---
title: MCP 协议入门：从 M×N 到 M+N，它到底解决了什么问题
feedId: 36261
source: 综合讨论
publishedAt: 2026-09-06
---

## 背景

过去一年做 Agent 开发的人大概率遇到过同一个场景：模型能力够了，但接数据、接工具很痛苦。每个 Agent 框架都有自己的 function calling 格式，每个 SaaS、每个数据库都要单独写对接代码。有 M 个应用、N 个数据源，就要写 M×N 个连接器，而且彼此不通用。

2024 年底 Anthropic 开源了 Model Context Protocol（MCP），目标就是把这层标准化：统一模型与外部工具、数据源之间的通信方式。

## 它解决的核心问题

MCP 本质是一套基于 JSON-RPC 2.0 的协议，规定了三类原语：

- **Tools**：模型可以调用的操作（查数据库、发消息、跑脚本）
- **Resources**：可被读取的上下文数据（文件、表结构、文档）
- **Prompts**：预置的提示词模板

再配上统一的发现机制（客户端启动时拉取 server 的能力列表）和两种传输方式（本地 stdio、远程 Streamable HTTP），M×N 的集成问题就变成了 M+N：应用方只需实现一次 client，工具方只需实现一次 server。

## 上手步骤

1. **理清角色**：Host 是运行模型的宿主（如 Claude Desktop 或你自己的 Agent），Client 负责与 server 通信，Server 暴露具体能力。
2. **选传输方式**：本地进程用 stdio，最简单；跨网络服务用 Streamable HTTP。
3. **写最小 server**：用官方 SDK（Python / TypeScript）注册一两个 tool，写清入参 schema 和描述。Python SDK 下三十行左右就能跑通一个读本地文件的 server。
4. **用 Inspector 验证**：先用 MCP Inspector 确认工具能被发现、调用、正常返回，再接入 Host。
5. **接入 Agent 循环**：把 server 返回的工具列表转成模型的 tool 定义，处理调用结果回填。

## 踩坑点

- **stdout 污染**：stdio 传输下，server 里任何 `print()` 或调试输出都会混进 JSON-RPC 流，直接导致协议解析崩溃。日志一律走 stderr 或写文件。
- **工具描述敷衍**：模型选工具、填参数几乎完全依赖 description 和 schema。描述含糊，模型就会瞎猜参数——这是"接上了但不好用"的最常见原因。
- **工具列表膨胀**：挂几十个工具后，工具定义本身占用大量上下文，模型选择准确率明显下降。按领域拆分 server，按需启用。
- **schema 太宽松**：参数全设成 optional、类型不收紧，等于把校验成本转嫁给模型。
- **危险操作没有确认层**：删除、支付、外发邮件这类工具，Host 侧必须加 human-in-the-loop，协议本身不替你做安全。

## 可复用建议

- 一个 server 只做一个领域；工具粒度细，但单 server 数量控制在 10 个以内。
- 把 tool description 当成"写给模型的 API 文档"来写：做什么、什么时候用、参数含义、失败时返回什么。
- 错误信息面向模型可读：返回结构化的错误描述而非裸异常，模型才有机会自我修正重试。
- server 尽量无状态，方便扩展和热更新。
- 调试分层：先 Inspector，再 Host，不要一上来就在 Agent 链路里排查。

## 总结

MCP 解决的不是模型能力问题，而是集成标准化问题。它的价值在于工具生态可组合、可复用，应用与工具两端解耦。它不会让你的 Agent 变聪明，但能把"给 Agent 接一个新工具"从两天降到两小时——对做自动化和插件的团队来说，这个杠杆值得投入。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/53884c43047eacb1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/bd184d612f5ac3e3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/a880e011f4999a0e.png)

