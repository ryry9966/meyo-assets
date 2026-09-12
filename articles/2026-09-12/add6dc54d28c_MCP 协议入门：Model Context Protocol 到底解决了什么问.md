---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 37189
source: 综合讨论
publishedAt: 2026-09-12
---

# 背景

做 Agent 开发经常遇到一个场景：模型本身能力没问题，卡在"接工具"上。想让它读写本地文件、查数据库、调内部 API，每接一个就要写一套 function calling schema、一套鉴权、一套错误处理。换个宿主框架，这些代码基本要重写一遍。

这就是 MCP（Model Context Protocol）要解决的背景问题。它由 Anthropic 于 2024 年底开源，定位很克制：一套让 LLM 应用与外部工具、数据源通信的开放协议。可以粗略理解为"AI 应用的 USB-C 接口"。

# 之前的问题到底是什么

核心是组合爆炸。假设有 M 个 AI 应用、N 个工具/数据源，点对点集成的成本是 M×N，而且每条边都要重复解决同一批问题：

1. **工具描述格式不统一**：各家 function calling schema 细节都有差异；
2. **鉴权各搞各的**：同一个 GitHub 工具，应用 A 用 PAT，应用 B 用 OAuth，应用 C 干脆内置；
3. **能力发现靠硬编码**：新加一个工具，宿主应用要发版才能用上；
4. **上下文传递没有规范**：工具返回的数据怎么裁剪、怎么分页，全靠各应用自己拍脑袋。

MCP 的一句话总结就是把这个成本从 M×N 压到 M+N：工具方只需实现一次 MCP Server，任何支持 MCP 的客户端都能直接用。

# MCP 的做法

协议基于 JSON-RPC 2.0，架构是标准的 Client-Server：

- **Host**：你的 Agent 应用，内部跑 MCP Client；
- **MCP Server**：独立进程，暴露三种原语，Tools（模型可调用的动作）、Resources（可读取的数据）、Prompts（预置提示模板）；
- **传输层**：本地用 stdio，远程用 Streamable HTTP，远程场景走 OAuth 2.1 鉴权。

# 上手步骤

1. **选 SDK**：官方有 TypeScript 和 Python SDK，Python 版 `pip install mcp` 即可；
2. **写最小 stdio Server**：用 `@mcp.tool()` 装饰器暴露一两个函数，比如查询内部服务状态；
3. **单独调试**：先用官方 Inspector（`npx @modelcontextprotocol/inspector`）验证工具能被发现、被调用，再接入 Agent；
4. **再考虑远程化**：确认确实需要多客户端共享后，才迁移到 HTTP + OAuth，别一上来就做远程。

# 踩坑点

- **工具描述质量决定调用成功率**：模型是靠 description 选工具的。写得含糊，模型要么不调用，要么乱传参数。把它当成写给初级工程师的 API 文档来写。
- **返回值撑爆上下文**：一个工具直接吐 5000 行 JSON，几轮对话就把窗口吃光。裁剪、分页、摘要要放在 Server 端做。
- **错误处理要"给模型看的"**：抛堆栈没有意义，返回结构化的错误信息和补救建议，模型才有自我修正的余地。
- **提示注入风险真实存在**：工具返回内容会进入模型上下文，外部数据源里的恶意指令会被当真。高危工具务必加人工确认环节。
- **SDK 版本协商问题**：协议演进很快，Client 和 Server 的 SDK 版本差异会导致 initialize 握手失败。排查时先对齐版本，再看别的。

# 可复用建议

- **工具粒度宁粗勿细**：与其暴露 20 个细粒度接口，不如封装 5 个贴合业务动作的工具，降低模型的决策负担；
- **接 Agent 前先用脚本压测 Server**：Inspector 或裸 JSON-RPC 脚本跑通再上 Agent，排障半径小很多；
- **给每次工具调用留日志**：参数、返回、耗时全记下来。经验上 90% 的"模型不干活"，最后查出来是 Server 侧报错。

# 总结

MCP 解决的不是模型能力问题，而是集成工程问题：把 M×N 的适配成本压到 M+N。它不神秘，本质是一套带能力发现和标准传输的 RPC 规范。如果你的 Agent 只接一两个工具，手写 function calling 足够；一旦工具数量和宿主应用开始增长，标准化的收益就会很快显现。建议路径：从本地 stdio Server 起步，跑通一个真实业务工具，再评估是否值得远程化。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/3496106a047d4465.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/25d2da661209344e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/9db701543f3bad44.png)

