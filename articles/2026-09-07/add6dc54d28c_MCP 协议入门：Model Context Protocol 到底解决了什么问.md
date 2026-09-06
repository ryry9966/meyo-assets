---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 36361
source: 综合讨论
publishedAt: 2026-09-07
---

## 背景

写 Agent 自动化的人大概都经历过同一个阶段：先靠 function calling 跑通，然后为每个数据源写胶水代码——GitHub 一套、数据库一套、内部 API 又一套。换一个 Agent 框架，这些连接器基本要重写。

MCP（Model Context Protocol）是 2024 年底 Anthropic 开源、之后被多家厂商跟进的开放协议，目标是把「模型/Agent 如何连接外部工具和数据」这件事标准化。社区常用「AI 应用的 USB-C」来形容它，宣传味偏重，但从工程角度看，方向是对的。

## 它到底解决什么问题

核心是把 N×M 的集成矩阵降成 N+M：

- N 个 Agent 宿主，M 个工具/数据源
- 没有 MCP：每个宿主要为每个数据源写一份定制连接器，鉴权、传输、schema 全是私有的
- 有了 MCP：数据源实现一次 MCP Server，宿主实现一次 MCP Client，双方按协议对话

MCP 定义了三类能力原语：**Tools**（模型主动调用的动作）、**Resources**（宿主侧可控的上下文数据）、**Prompts**（模板化交互）。传输层主流两种：本地进程走 stdio，远程服务走 Streamable HTTP，消息格式基于 JSON-RPC 2.0。

对 OpenClaw 这类可插拔的 Agent 运行时，价值很直接：工具生态不再绑定单一框架，Server 写一次，多端复用。

## 实践步骤

1. **先别自己写 Server。** 从现成的跑起：filesystem、fetch、各类数据库的官方或社区 Server 都有。第一目标是验证链路，不是造轮子。
2. **选传输方式。** 本地可信工具用 stdio，简单直接；需要共享给团队或跨机部署，再上 Streamable HTTP。
3. **接入运行时。** 在 OpenClaw 里注册 MCP Client，把 Server 暴露的 tools 挂进 Agent 的工具列表。
4. **小闭环验证。** 给 Agent 一个明确任务，观察它是否选对工具、参数是否正确。日志里记录每次 tool call 的入参和耗时。
5. **确认没有现成轮子后，再写自己的 Server。** 从一两个工具起步，input schema 用 JSON Schema 写清楚，description 写明「什么时候该用、什么时候不该用」。

## 踩坑点

- **description 决定调用准确率。** 描述模糊，模型就会选错工具或传错参数——这不是模型的问题，是你的文档没写好。
- **工具数量失控。** 挂 30+ 个工具，上下文膨胀且选择混乱。按场景拆分 Server，按需加载。
- **MCP 不帮你解决鉴权。** stdio Server 的密钥走环境变量，别提交进仓库；远程 Server 的 OAuth 要自己接。
- **stdio 子进程崩了连接就断。** 宿主侧要有重启和健康检查逻辑，否则任务跑到一半静默失败。
- **协议和 SDK 在演进。** 比如 HTTP+SSE 已被 Streamable HTTP 取代，锁版本，升级前看 changelog。
- **安全别掉以轻心。** 工具返回内容会进入 Agent 上下文，间接注入风险真实存在；文件系统类 Server 别给根目录权限，坚持最小授权。

## 可复用建议

- 把 MCP Server 当微服务依赖管理：锁版本、健康检查、结构化日志，一个 Server 只管一件事。
- description 当作写给新同事的说明书：用途边界、参数示例、典型错误场景。
- 每次 tool call 记录结构化日志，排障效率差一个数量级。

## 总结

MCP 解决的是「接线」的标准化问题，把 N×M 降成 N+M，工具生态第一次具备了跨框架复用的可能。但它不解决 Agent 的可靠性——schema 设计、描述质量、权限控制、注入防护，这些仍然是你自己的工程问题。协议管插口，质量管体验。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/d3dc5e08cc1f86b6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/f876f5471767ea25.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/c2939b9ac556522b.png)

