---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 36093
source: 综合讨论
publishedAt: 2026-09-05
---

## 背景

Agent 要真正干活，光有模型不够，还得能碰到真实世界：读文件、查数据库、调内部 API。OpenClaw 这类自动化 Agent 的核心价值，就建立在"模型 + 工具"的组合上。

MCP（Model Context Protocol）是 Anthropic 在 2024 年底开源的协议，目标很明确：给"模型如何连接外部工具和数据"定一个统一标准。社区常把它类比成 AI 应用的 USB-C——接口统一了，设备随便换。

## 它到底解决了什么问题

在 MCP 之前，这是典型的 M×N 问题：

- 你有 M 个 Agent 宿主（OpenClaw、Claude Desktop、自研 Agent……）
- 你有 N 个数据源和工具（GitHub、Postgres、内部系统……）

没有标准，每对组合都要写一次胶水代码：工具怎么描述给模型、调用怎么传参、鉴权怎么走，全靠各家自定义。结果就是同一个 GitHub 工具，A 平台写一份插件，B 平台再写一份，参数格式还互不兼容。

MCP 把 M×N 变成 M+N：工具方只需实现一次 MCP Server，任何支持协议的宿主都能即插即用。协议本身规定了三件事：

1. **发现**：宿主启动时向 Server 拉取工具列表，每个工具带名称、描述和 JSON Schema 入参定义，模型据此决定何时调用。
2. **调用**：宿主把模型的调用意图按协议转发给 Server，Server 执行后返回结构化结果。
3. **上下文**：除 tools 外还有 resources（数据读取）和 prompts（模板）两类原语，把"给模型看的内容"和"给用户看的内容"分开管理。

## 实际接入步骤

以"让 Agent 查本地 PostgreSQL"为例：

1. **选 Server**：优先找现成的（官方和社区维护了一批 filesystem、GitHub、Postgres 等），别急着自己写。
2. **配置宿主**：在 OpenClaw 的 MCP 配置里注册，本地进程用 stdio 传输（写清 command 和 args），远程服务用 Streamable HTTP。
3. **验证发现**：启动后确认工具列表被正确拉取，用 MCP Inspector 或日志检查每个工具的 schema。
4. **小步试调**：先用只读工具跑通一轮完整调用链，再开放写操作。
5. **收权限**：给 Server 单独账号、最小权限，删改类操作加确认门槛。

## 踩坑点

- **stdio 模式下别往 stdout 打日志**。stdout 是协议通道，一行多余输出就可能让宿主解析失败，日志一律走 stderr。这是新手最高频的问题。
- **工具描述是写给模型看的，不是写给人看的**。描述含糊，模型就会选错工具或乱传参，宁可啰嗦也要把适用场景和参数含义写清。
- **协议版本在演进**：早期教程基于 SSE 传输，新版本已转向 Streamable HTTP，照抄旧配置会连不上，注意 SDK 版本对齐。
- **上下文膨胀**：一个工具返回整张表，几轮对话就把窗口吃满，Server 侧要做好分页和截断。
- **第三方 Server 别裸跑**：工具返回内容会进入模型上下文，等于多了一条注入路径，来源不明的先审代码再上线。

## 可复用建议

- 一个 Server 管一个领域，工具保持"小而明确"，用动词命名（`query_order` 而不是 `process`）。
- 读写分离：查询工具可以默认开放，写/删工具默认关闭、按需开启。
- 调试三板斧：MCP Inspector 看协议交互、stderr 日志看执行细节、单测覆盖入参 schema。

## 总结

MCP 解决的不是"模型不够聪明"，而是"工具接入没有标准"这个工程问题。它把每次集成从定制开发变成配置工作，对同时维护多个 Agent、多个数据源的团队是实打实的成本下降。在 OpenClaw 的实践中，建议从现成 Server 起步、以最小权限试跑、把工具描述当提示词来打磨——协议只是壳，工具设计的质量才决定 Agent 的上限。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/c0ac41c18cddf4ac.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/940ed3e59e493f36.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/65f5db7d10c2f605.png)

