---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 37201
source: 综合讨论
publishedAt: 2026-09-12
---

## 背景

过去一年做 Agent 的同学大概都有类似体验：模型能力已经够用，瓶颈反而卡在"接工具"上。每个 Agent 框架都定义了一套自己的插件/工具格式，每个数据源——GitHub、数据库、本地文件、内部 API——都要按框架规范单独封装一遍。你写了三个 Agent，同样的 GitHub 集成就得写三次。

## MCP 解决的核心问题

MCP（Model Context Protocol）是 Anthropic 在 2024 年底开源的协议，一句话概括：**把"模型应用"和"外部能力"之间的连接方式标准化**。

本质是经典的 M×N 问题。没有 MCP 时，M 个应用对接 N 个工具，需要 M×N 份胶水代码；有了 MCP，工具方实现一次 MCP Server，应用方实现一次 MCP Client，复杂度降为 M+N。可以类比为 USB-C 之于外设、LSP 之于编辑器。

协议层面它约定了三件事：

- **传输**：本地用 stdio，远程用 Streamable HTTP（早期是 SSE，新 spec 已替换）；
- **消息**：JSON-RPC 2.0；
- **能力原语**：Tools（模型可调用）、Resources（可读取的上下文）、Prompts（预置模板）。

## 上手步骤

1. 先跑现成 Server 找体感，比如 filesystem 或 fetch；
2. 在 MCP Client（OpenClaw 等）的配置里注册该 server，声明启动命令；
3. 重启后确认工具列表被注入；
4. 用 `npx @modelcontextprotocol/inspector` 打开调试面板，能看到消息级请求/响应，比翻日志高效得多；
5. 自研时用官方 SDK（Python/TS）以装饰器声明工具函数，主要工作量在工具描述而非代码本身。

## 踩坑点

- **stdout 污染**：stdio 模式下，Server 往 stdout 打调试日志会直接破坏 JSON-RPC 流，日志一律走 stderr。新手第一大坑。
- **工具描述就是提示词**：模型靠 description 决定何时调用、怎么传参。参数说明含糊，调用就会随机失败，且很难从报错看出原因。
- **上下文膨胀**：一次挂几十个 server、上百个工具，光 schema 就吃掉大量 token，还会降低选择准确率。按需启用，别贪多。
- **安全边界**：挂一个第三方 MCP Server，等于授权它代表你操作数据。本地 server 检查读写路径范围，远程 server 关注鉴权方式，不要无脑放行。
- **spec 迭代快**：大量旧教程仍停留在 SSE 传输，新版本已是 Streamable HTTP，配置报错时先核对协议版本。

## 可复用建议

- 优先复用现成 Server，只有内部系统才值得自研；
- 工具设计偏"粗粒度业务动作"而非"细粒度 API 包装"，`create_issue` 比 `post_json_to_endpoint` 好用得多；
- 本地开发全走 stdio + inspector，跑通了再考虑远程部署；
- 把每个 server 当作独立的供应链依赖管理，记录来源与版本。

## 总结

MCP 没有让 Agent 变聪明，它只是把集成成本从 O(M×N) 降到 O(M+N)，并顺手统一了传输、发现和描述格式。对个人开发者，意味着一次封装处处可用；对团队，意味着工具层和应用层可以分开演进。如果你在做自动化或插件生态，值得花一个下午把第一个 server 跑起来——理解协议最好的方式，就是亲手接一次。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/80af67ab7fcb7021.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/4a422a1c5294c586.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/c208be742d1c400b.png)

