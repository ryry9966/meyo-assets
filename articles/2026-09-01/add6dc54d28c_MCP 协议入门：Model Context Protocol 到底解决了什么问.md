---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 35647
source: 综合讨论
publishedAt: 2026-09-01
---

# MCP 协议入门：Model Context Protocol 到底解决了什么问题

## 背景：Agent 工具链的集成成本

在 OpenClaw、Agent、插件和自动化实践里，一个高频痛点不是“模型不够聪明”，而是“让模型能用上工具”这件事太琐碎。

早期的常见做法是：

- 给每个工具写一个插件或适配器；
- 自己定义 HTTP API，再在 Agent 侧写 function calling schema；
- 每个平台、每个模型客户端的工具描述格式都不同；
- 工具返回内容直接塞进 prompt，容易出现上下文膨胀、解析失败、错误难追溯。

当工具只有一两个时，胶水代码还可以接受。但一旦接入数据库、文件系统、浏览器、Shell、内部 API、知识库、消息通知等十几个能力时，维护成本会迅速变成 `N × M` 问题：每种工具都要适配每种 Agent 客户端。

MCP 要解决的，正是这个“标准化接入”问题。

## 问题：不是智能问题，是接口问题

MCP 全称 Model Context Protocol，核心目标不是增强模型推理，而是做一件事：

> 为模型和外部工具/数据源之间提供一套统一的上下文交换协议。

它主要解决四类问题：

1. **工具发现与描述不一致**：不同工具的 schema、参数说明、返回格式各自为政。
2. **调用方式不统一**：有的是 HTTP，有的是 SDK，有的是本地脚本。
3. **上下文传递混乱**：工具返回内容无法结构化，模型难以判断成功/失败、该继续还是停止。
4. **治理与复用困难**：权限、日志、超时、重试、错误处理分散在各个胶水代码里。

MCP 把这些抽象成三个核心概念：

- **Tools**：模型可以调用的操作；
- **Resources**：模型可以读取的数据，如文件、文档、数据库记录；
- **Prompts**：可复用的提示模板。

底层消息基于 JSON-RPC 2.0，传输层支持 stdio、SSE 和 Streamable HTTP。客户端与服务器之间会做能力协商，各自声明支持哪些方法。

## 做法：一个最小 MCP Server 的接入步骤

下面是一个工程化场景，比如让 OpenClaw 通过 MCP 调用内部天气 API。

### 1. 先定义工具 schema

不要先写 handler，先写 JSON Schema。模型对工具的调用质量，很大程度上取决于 description 是否具体。

```json
{
  "name": "get_weather",
  "description": "Get current weather for a city. Use only when the user asks for live weather.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "city": { "type": "string", "description": "City name, e.g. Shanghai" }
    },
    "required": ["city"]
  }
}
```

### 2. 实现 handler，返回结构化内容

MCP server 的 handler 返回 content 数组。建议在 text 里放精简 JSON 摘要，而不是把完整响应直接倒给模型。

```json
{
  "content": [
    {
      "type": "text",
      "text": "{\"ok\":true,\"data\":{\"temp\":18,\"condition\":\"cloudy\"},\"hints\":[]}"
    }
  ]
}
```

### 3. 启动 server 并注册

如果是 stdio transport，启动命令通过标准输入输出通信。注册到客户端时配置类似：

```json
{
  "mcpServers": {
    "weather": {
      "command": "node",
      "args": ["/path/to/weather-server.js"]
    }
  }
}
```

### 4. 联调

优先使用 MCP Inspector 或客户端内置工具列表，确认三件事：

- 工具能被列出；
- 参数 schema 能被模型正确理解；
- 返回内容不会污染上下文。

再跑一组固定测试 query，例如“北京现在多少度”，观察模型是否会调用工具、解析结果、给出最终回答。

## 踩坑点

### 1. description 写得太宽泛

模型不是通过代码逻辑判断工具，而是通过自然语言和 schema 选择工具。如果 description 只写“获取天气”，没有写“仅在用户询问实时天气时使用”，模型可能会在奇怪的场景调用它。

### 2. 返回内容直接塞满上下文

工具返回 5000 行 HTML 或完整 API 响应，会迅速消耗 token，还容易让模型忽略关键字段。应该让工具返回精简摘要，把大文件、长文档转成 resource URI，按需二次读取。

### 3. stdio 下日志打到 stdout

这是最容易踩的坑。MCP 的 JSON-RPC 消息走 stdout，如果 server 里 `console.log` 直接输出日志，会破坏协议消息。日志必须打到 stderr。

### 4. 能力协商不一致

server 声明了某个 tool，但 handler 没实现；或者 client 不支持某种传输层，联调时表现为“工具列表为空”或“调用无响应”。先检查版本和 capabilities。

### 5. 长耗时调用没有超时/取消

如果工具要执行 Shell 命令或访问外部 API，却没有超时、取消、重试机制，整个 Agent 循环会被卡住。MCP 本身不解决这个问题，需要在 server 实现里做。

### 6. 权限边界被忽略

MCP server 可能拥有文件系统、Shell、数据库权限。接入第三方 server 前，要确认它有最小权限，并尽量在客户端限制可用工具列表。

## 可复用建议

- **无状态、幂等优先**：同一个工具在同样输入下尽量产出稳定结果；有副作用的操作提供 `dry_run` 参数。
- **统一返回 envelope**：`{ok, data, error, hints}`，让模型容易判断成功与失败。
- **单一职责**：一个 MCP server 只做一类能力，例如“文件操作”“天气查询”“内部 API”；复杂流程拆成多个 server。
- **固定测试集**：把 10-20 条典型 tool call 固化成回归用例，升级 server 或换模型时跑一遍。
- **在 OpenClaw 侧注入使用约定**：通过规则或系统提示说明“优先使用 MCP 工具”“工具返回 error 时停止并报告”，比完全依赖模型自觉更稳定。
- **不要为了 MCP 而 MCP**：如果只有一两个工具、只在一个客户端用，普通 function calling 轻量够用。当工具数量超过 3 个、需要跨客户端复用，或要把工具能力开放给多个 Agent 时，再引入 MCP 更划算。

## 总结

MCP 并不提升模型的推理能力，它做的是把 Agent 与外部工具/数据之间的“接口层”标准化。对工程化实践来说，最大的收益是：

- 减少重复适配；
- 让工具可发现、可组合、可治理；
- 让工具返回内容更可控，降低上下文污染；
- 让团队能沉淀一套可复用的 MCP server，而不是每次重新接线。

如果你的 Agent 项目已经开始出现“工具适配代码比业务逻辑还多”的迹象，MCP 是一个值得投入的工程方向。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/8b9e0755a4c4fdc5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/19126ba922318d72.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/3f115e47d746403e.png)

