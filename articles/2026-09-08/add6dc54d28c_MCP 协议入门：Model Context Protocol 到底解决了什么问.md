---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 36628
source: 综合讨论
publishedAt: 2026-09-08
---

## 背景

过去两年做 Agent 的人都会遇到同一个场景：模型能力已经够用，卡住的是“接数据、调工具”。文件系统、数据库、内部 API、浏览器……每个应用接每个工具，都要写一份定制的胶水代码。

MCP（Model Context Protocol）是 Anthropic 在 2024 年底开源的协议，目标就是把这件事标准化。它不是新模型，也不是框架，而是一层约定：AI 应用如何发现并调用外部工具与数据。

## 问题：N×M 的集成成本

没有标准之前，M 个 AI 应用要接 N 个工具，最坏要写 M×N 个连接器，各家的鉴权、参数格式、错误处理全不一样。对工具提供方同样痛苦：写好一个内部服务，想给不同 Agent 用，就得为每个宿主各写一份适配。

MCP 把它压成 N+M：工具方实现一次 Server，应用方实现一次 Client，中间用统一协议对话。

## 核心模型：一句话说清

MCP 是基于 JSON-RPC 的客户端-服务端协议：

- **Host/Client**：Agent 或桌面应用一侧，负责连接并管理 Server；
- **Server**：暴露三类能力——Tools（模型可调用的动作）、Resources（可读取的数据）、Prompts（预置模板）；
- **传输层**：本机常用 stdio，远程用 Streamable HTTP。

注意：选哪个工具、怎么拼参数，决策始终由模型做。MCP 只规范“发现”和“调用”这两步，不替你做编排。

## 在 OpenClaw 里接入的最小路径

1. **明确场景**：先确认 Agent 缺什么能力（查库、发消息、读文件），别一上来就“接一切”。
2. **优先复用**：常见需求（文件、Git、浏览器）先找现成的开源 MCP Server。
3. **选传输**：本机进程用 stdio 最简单；跨机器或需常驻服务，用 Streamable HTTP。
4. **定义工具**：JSON Schema 声明参数，字段尽量扁平，description 写给模型看，不是写给人看。
5. **用 MCP Inspector 单测**：官方调试工具可脱离宿主直接试调用，先跑通再接入。
6. **挂进 OpenClaw 配置**，观察工具调用日志，确认模型选对了工具。

## 踩坑点

- **stdout 污染**：stdio 模式下 Server 往 stdout 打日志会直接破坏协议流，日志必须走 stderr。新手第一大坑。
- **工具描述太弱**：模型靠 description 选工具。写“处理数据”这种模糊描述，模型大概率选错或不敢选。要写清何时该用、何时不用。
- **Schema 过度嵌套**：深层嵌套对象、大量必填参数会让参数生成质量骤降。保持扁平，能少一个字段就少一个。
- **返回值过大**：一次查表返回几千行，直接撑爆上下文。Server 侧要做截断、分页和摘要。
- **没有超时和取消**：工具卡死会拖住整个 Agent 循环，务必设超时并支持取消。

## 可复用的建议

- 工具按 Agent 工作流设计，而不是内部 API 一比一映射。宁可多个小工具，也别做一个带十种模式的巨型工具。
- 一个 Server 只管一个领域，工具面小而清晰。
- 把 description 当 prompt 写，这是性价比最高的优化。
- 错误信息返回给模型看，让它能自我修正，而不是只抛异常就断掉。

## 总结

MCP 本身没有魔法，本质是 JSON-RPC 加一套 Schema 与描述规范。它真正的价值在于把 M×N 的集成问题压缩成 N+M，让工具生态可以被复用。对做自动化的团队来说，关键不在“用了 MCP”，而在于工具描述、Schema 设计和错误处理是否真的对模型友好——这部分工程活，协议替代不了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/c440a5468b38b874.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/b0bacf7f2c187bb2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/91921948beed242d.png)

