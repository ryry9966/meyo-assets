---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 36885
source: 综合讨论
publishedAt: 2026-09-10
---

## 背景

过去一年做 Agent 自动化的人基本都撞上同一个问题：模型本身能力不差，但它是个“封闭的大脑”——读不到本地文件、查不了内部数据库、调不了公司 API。于是大家各自造轮子：这个框架一套插件格式，那个客户端另一套，互不相通。

2024 年底 Anthropic 开源了 MCP（Model Context Protocol），定位很朴素：给“模型 ↔ 外部工具/数据”定一个标准接口。可以理解为 AI 时代的 USB-C。

## 它到底解决了什么问题

核心是两件事：

**1. M×N 的集成问题。** M 个模型/Agent 框架 × N 个工具，传统做法要写 M×N 个适配器。有了 MCP，工具方按协议实现一次 MCP Server，任何兼容的 Host 都能直接接入，集成成本从 M×N 降到 M+N。

**2. 上下文管道问题。** 工具怎么被发现、参数 schema 怎么描述、结果怎么回传，这些脏活以前每个框架自定义。MCP 标准化为三类原语：

- **Tools**：模型可调用的动作（如“查询订单”）
- **Resources**：可读取的数据上下文（文件、配置）
- **Prompts**：预置提示词模板

架构分三层：Host（你的 Agent 应用，如 OpenClaw）内嵌 MCP Client，Client 与 MCP Server 通信。本地场景走 stdio 传输，远程服务走 Streamable HTTP。

## 上手：四步跑通一个本地 Server

1. **确认 Host 支持 MCP**，在配置文件里找到 MCP servers 配置段（一般是 JSON）。
2. **先跑现成的 Server 验证链路**，比如官方 filesystem server：

```json
{
  "mcpServers": {
    "fs": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/Users/me/projects"]
    }
  }
}
```

3. **重启客户端，验证工具注入**：问一句“列出这个目录下的文件”，观察是否发起了 tool call。
4. **自己写一个**：用 Python SDK 约 30 行就能暴露自定义工具，核心是声明函数签名和描述，协议格式由 SDK 处理。

## 踩坑点

- **stdout 被占用。** stdio 模式下 stdout 是协议通道，在 Server 里随手 `print` 调试信息会直接打坏 JSON-RPC 流，日志必须走 stderr。新手第一大坑。
- **工具描述吃上下文。** 每个工具的名称+描述都会进 system prompt。挂十几个 Server、上百个工具后，token 烧得快，模型选错工具的概率也明显上升。
- **环境与路径。** npx/uvx 启动依赖 Node/Python 环境，路径写错或 env 没配时经常静默失败。先在终端手动跑一遍启动命令确认无误。
- **安全边界。** 不要随意运行来路不明的 MCP Server——工具结果会进入模型上下文，等于给提示注入开了口子。filesystem 类 Server 务必收窄目录范围，写操作默认要求人工确认。

## 可复用建议

- 工具命名用动词+对象（`search_tickets` 而非 `tool1`），描述写清“什么场景该用我”，模型就靠这两样做选择。
- 单个 Server 的工具数控制在 10 个以内，做能力聚合，别把内部 API 的几十个 endpoint 原样全暴露。
- 返回结果做裁剪：给模型摘要和结构化字段，原始大 JSON 截断或落盘后给引用。
- 用 uvx/npx 固定版本号，避免上游更新悄悄改坏 schema。

## 总结

MCP 没有让模型变聪明，它解决的是工程问题：把“接工具”从重复劳动变成标准件。判断一个能力该不该包成 MCP Server，标准很简单——它是否会被多个客户端、多个场景复用。是，就按协议写一次；只是单场景的一次性脚本，直接写函数调用反而更省事。工具做薄、描述做准、权限收窄，是我在实际接入中最有效的三条经验。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/a3d9805e3b38cc53.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/c8a523b859cb69c6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/5e4ce713f157c872.png)

