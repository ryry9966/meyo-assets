---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 34588
source: 综合讨论
publishedAt: 2026-08-25
---

## 背景：Agent 工具接入的重复劳动

在 OpenClaw 这类 Agent 项目里，工具接入往往是最容易失控的部分。早期做法是给每个外部能力写一个 adapter：数据库查询、文件读取、浏览器操作、HTTP 接口，各自定义 JSON 结构、错误格式、结果裁剪逻辑。换一个客户端或复用同一个工具时，又要重写一遍。

更麻烦的是，为了“教会”模型使用工具，我们常把工具说明、参数示例、返回格式全部塞进 system prompt。工具一多，上下文膨胀，模型注意力被稀释，维护成本直线上升。

MCP（Model Context Protocol）要解决的正是这个集成问题：它不是让模型变强，而是把“工具、资源、提示”的接入方式标准化，减少 Agent 与外部能力之间的适配成本。

## 问题：没有统一协议会带来什么

**1. 工具复用难**

同一个 SQLite 查询能力，项目 A 用函数调用格式，项目 B 用 JSON-RPC，项目 C 又用自定义插件。每次迁移都要重新理解语义、重写 handler。

**2. 上下文供给失控**

文件内容、网页正文、数据库结果未经统一裁剪就进入上下文。模型要么被大量无关信息干扰，要么 token 消耗急剧上升。

**3. 多客户端维护成本高**

CLI、Web、Agent、自动化脚本各自接入工具，每次新增能力都要在不同端重复开发。

MCP 的设计是把这些能力抽象为三类原语：`tools`（可执行操作）、`resources`（可读取的数据）、`prompts`（可复用的提示模板）。Server 暴露能力，Client 负责发现、调用和结果消费。OpenClaw 作为 Client，只需要维护 server 配置，不需要关心每个工具的传输细节。

## 做法：在 OpenClaw 中接入一个 MCP server

以接入 SQLite MCP server 为例，配置如下：

```json
{
  "mcpServers": {
    "sqlite": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-sqlite", "/data/orders.db"]
    }
  }
}
```

启动后，Client 会通过 `initialize` 握手，然后拉取 `tools/list`。这时不要急着写复杂任务，先验证工具是否正常出现，再调用一个只读 SQL 查询，确认结果格式符合预期。

接入后，Agent 指令里不要再重复描述 SQL 工具的参数和返回格式。只需要说“使用 sqlite 工具查询订单表”。工具描述和 schema 由 server 提供，避免 prompt 膨胀。

返回结果必须裁剪。数据库查询可能返回几万行，建议在 server 层或 client 层设置 `maxRows`，或者让模型先 `count` 再分页。否则一次查询就能把上下文撑爆。

## 踩坑点

**stdio 日志污染**

很多 MCP server 通过 stdio 传输 JSON-RPC 消息，同时输出日志。如果 server 把日志打到 stdout，会混入协议消息，导致解析失败。遇到 `Unexpected token` 类错误，先检查 server 是否把日志写到了 stderr。

**schema 质量参差**

部分社区 server 的 `inputSchema` 只写 `type: object`，没有字段说明，模型不知道该怎么填参数。接入前先手动检查 schema，必要时用一个代理 server 补全参数描述。

**上下文爆炸**

文件读取和数据库查询是最容易踩的坑。一定要在 client 或 server 层做 `maxBytes` / `maxRows` 限制，或者提供分页工具，让模型按需取数。

**权限边界过宽**

filesystem server 如果挂载 `$HOME`，模型可能误读写用户文件。只开放项目工作目录，写操作工具加确认或 dry-run。

**版本不匹配**

MCP 协议还在演进，server 和 client 的 protocol version 不一致会导致握手失败。锁定 server 版本，升级前在测试环境验证。

## 可复用建议

- 把 MCP server 当作“能力边界”管理，不要在 prompt 里写死工具说明。
- 按只读 → 读写 → 外部服务顺序接入，先降低风险。
- 工具返回必须裁剪，错误信息结构化回传，让模型能自我纠正。
- 配置模板化，常用 server 存入项目仓库，团队复用。
- 记录每次工具调用的耗时和 token 消耗，定位隐藏成本。

## 总结

MCP 解决的是集成复杂度从 N×M 降到 N+M 的问题。它让 Agent 的上下文更干净，工具可跨项目复用，降低维护成本。但不要神化协议本身，实际收益取决于 server 质量、上下文裁剪策略和权限设计。对 OpenClaw 用户来说，先把只读、低风险的工具接起来，跑通“发现 → 调用 → 返回 → 裁剪”的闭环，再逐步扩大边界，是更务实的路径。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/a764e11c42448054.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/158f077197cbd18a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/bf089ccd6c6c4aea.png)

