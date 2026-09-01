---
title: MCP 协议入门：把 M×N 的集成问题变成 M+N
feedId: 35700
source: 综合讨论
publishedAt: 2026-09-01
---

## 背景

Agent 要干活，就得碰真实世界：读文件、查数据库、调内部 API。但在 MCP 出现之前，这件事没有标准做法——每个 Agent 框架自定义一套 tool schema，每个工具提供方为每个平台单独写适配。3 个客户端 × 10 个工具 = 30 份胶水代码，且互不通用。

MCP（Model Context Protocol）于 2024 年底开源，本质是一份基于 JSON-RPC 2.0 的约定：工具方实现一次 Server，任何支持 MCP 的客户端（Host）都能即插即用。目前主流 Agent 框架基本都已跟进。

## 它到底解决了什么

三个具体问题：

1. **集成成本从 M×N 降到 M+N**。工具开发者只面向协议写一次；客户端只需实现一遍 Client 侧。
2. **上下文注入标准化**。Tools（可调用函数）、Resources（可读数据）、Prompts（模板）三类能力有统一的发现与描述机制，Agent 运行时动态拉取，不用在代码里硬编码 schema。
3. **进程边界清晰**。Server 独立进程运行，崩了不拖垮 Host，权限和生命周期也好管理。

一句话：MCP 不让模型变聪明，它让"接入"这件事变得可组合。

## 在 OpenClaw 里跑通一个 Server

1. **选一个 Server**。建议从官方参考实现起步，比如 filesystem，走 stdio 传输，零网络依赖。
2. **写配置**。核心是 `command`、`args`、`env` 三项，例如用 npx 启动一个 Node 版 Server，并把允许访问的目录通过参数传入。
3. **重启客户端，确认工具列表被拉到**。日志里应能看到 `tools/list` 的返回结果。
4. **用只读工具做端到端验证**（如列出目录），观察模型是否正确传参、返回是否被正确解读。
5. **看上下文开销**。工具描述会整体注入提示，数一下 token，心里有底。

## 踩坑点

- **stdio 静默失败**。环境变量没传、PATH 不对、Windows 下要用 `cmd /c` 包一层启动 npx——症状都一样："工具列表为空"。先在终端手动跑一遍启动命令，再回头查配置。
- **协议版本漂移**。规范迭代很快：传输层从 HTTP+SSE 换成了 Streamable HTTP，鉴权方案也调整过。照旧教程抄配置连不上时，先核对规范版本和客户端支持情况。
- **token 失控**。挂十几个 Server、每个暴露几十个工具，光工具描述就能吃掉上万 token。按需启用，能关就关。
- **安全别裸奔**。Server 以你的用户权限运行，工具返回内容里可能被注入指令，诱导模型执行危险操作。写操作务必加人工确认，生产环境优先只读工具。

## 可复用建议

- 自研 Server 保持单一职责，一个领域一个 Server，工具数控制在个位数。
- 工具的 description 是写给模型看的：参数语义写清楚，比堆示例更省 token。
- 多 Server 场景给工具名加前缀，避免命名冲突。
- 给工具调用留日志，出问题时能回放定位。

## 总结

MCP 做的事很朴素：给"模型连工具"定一个 USB-C 口。对 OpenClaw 这类自动化实践者，价值在于把适配层的工作外包给了协议——你的精力可以回到真正重要的事上：选对工具、管好权限、控住上下文。先跑通一个最小 Server，再谈扩展。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/dff4dd921a5043fa.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/85dbc64ebfd4c6f9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/3069408b4dc959a3.png)

