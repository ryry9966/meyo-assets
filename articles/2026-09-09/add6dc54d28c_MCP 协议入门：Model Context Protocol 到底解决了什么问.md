---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 36700
source: 综合讨论
publishedAt: 2026-09-09
---

## 背景

让 LLM 操作外部世界，过去基本靠两条路：Function Calling 和各家自己定义的插件格式。Function Calling 解决了"模型怎么表达要调用什么"，但没解决"这些工具从哪来、怎么接"。于是每个 Agent 框架、每个客户端都在重复造轮子：接 GitHub 写一套，接数据库写一套，换个宿主应用又得重写——典型的 M×N 集成成本问题。

2024 年底 Anthropic 开源了 Model Context Protocol（MCP），把"模型侧"和"工具侧"之间的接口标准化。目前它已经是事实上的行业标准之一，OpenClaw 生态里的不少插件和自动化流水线也在往这个方向收敛。

## 问题：MCP 到底解决什么

一句话：**把 M×N 的集成问题降成 M+N**。

工具开发者只需实现一次 MCP Server，任何支持 MCP 的宿主都能直接用；宿主只需实现一次 MCP Client，就能接入所有 Server。协议本身很朴素：

- 基于 JSON-RPC 2.0，支持 stdio 和 Streamable HTTP 两种传输
- 连接建立时做能力协商
- Server 对外暴露三类原语：**Tools**（模型主动调用的动作）、**Resources**（应用可读取的数据）、**Prompts**（预置提示词模板）

注意边界：MCP 不解决鉴权、计费、工具分发这些上层问题，它只管消息格式和交互流程，鉴权仍在传输层自己做。

## 做法：跑通一个最小链路

以接入本地文件系统 server 为例：

1. 在宿主的 Agent 配置中声明 MCP server，stdio 方式只需给出启动命令和参数：

```json
{
  "mcpServers": {
    "fs": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/dir"]
    }
  }
}
```

2. 启动宿主，从日志确认 `list_tools` 握手完成、工具已注入。
3. 用官方 Inspector（`npx @modelcontextprotocol/inspector`）绕过宿主单独调试 server，先确认 server 本身是好的，再排查宿主配置。
4. 给 Agent 一个真实任务，验证它能选对工具、传对参数。

自己写 server 的话，官方 SDK（Python/TypeScript）十几行就能把一个函数包装成 tool，真正花时间的部分是 description 和 JSON Schema。

## 踩坑点

- **工具描述质量决定调用质量。** 模型选工具完全依赖 name + description + schema，写得含糊就会出现"该调不调、不该调乱调"。
- **工具数量别失控。** 几十个工具同时注入，选择准确率明显下滑，token 消耗也大，按需启用。
- **安全别想当然。** server 返回值会直接进模型上下文，恶意数据可借 prompt injection 诱导模型调用写操作。接第三方 server 前先读源码，至少先以只读模式跑。
- **传输方式注意版本。** 早期文档里的 HTTP+SSE 已被 Streamable HTTP 取代，照抄老教程容易对不上。
- **stdio 模式的进程由宿主管。** Windows 下 `npx.cmd` 的解析问题、子进程残留问题都实际遇到过。
- **日志。** MCP 报错经常被宿主吞掉，定位问题直接看 Inspector 或 stderr。

## 可复用建议

- 优先用现成的官方/社区 server，只有内部系统才值得自己写。
- 自己写时遵守"小而清晰"：一个 tool 干一件事，参数用扁平 schema，有副作用的操作加确认机制。
- 把读操作和写操作拆成不同 server，方便权限隔离和灰度。
- 工具描述当产品文档写，不要当代码注释写。
- 上线前用 Inspector 做一遍协议级回归，别只靠宿主端验证。

## 总结

MCP 的价值不在技术复杂度——协议本身相当朴素——而在于把"工具接入"从每次重新发明，变成了一次标准化。对做 Agent 和自动化的同学，建议花半天把最小链路跑通，之后接任何新工具都只是增量工作。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/14ba7edc68f7d21b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/a5afcf6ec6e08499.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/ed0a6b4b3c7f3310.png)

