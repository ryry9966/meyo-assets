---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 37126
source: 综合讨论
publishedAt: 2026-09-12
---

## 背景

给 Agent 接外部能力这件事，过去两年基本是各写各的：OpenAI 的 function calling、LangChain 的 Tool、各家自研的插件格式，描述方式、调用约定、错误处理都不一样。2024 年底 Anthropic 开源了 Model Context Protocol（MCP），试图把"模型怎么用工具、怎么读上下文"标准化。它本身只是协议，不绑定任何模型厂商，Client 和 Server 都可以自由实现。

## 它到底解决什么问题

核心是两点：

**1. M×N 的集成爆炸。** 没有统一协议时，M 个 Agent 应用要接 N 个数据源/工具，最坏要写 M×N 份胶水代码。MCP 把它变成 M+N：工具方写一次 MCP Server，应用方实现一次 MCP Client，双方按 JSON-RPC 约定通信。在 OpenClaw 里接一个新 Server，成本接近改一段配置，而不是写一个插件。

**2. 能力发现没有标准。** 传统 function calling 的工具列表是启动时硬编码的。MCP Client 启动时向 Server 发 `tools/list`，动态拿到工具名、参数 schema 和描述，Agent 可以在运行时挂载、卸载能力。

协议里有三个原语，容易混但值得记牢：

- **Tools**：模型主动调用，有副作用（写文件、发请求）；
- **Resources**：应用侧读取的上下文（文件、表数据），由宿主决定注入什么；
- **Prompts**：用户触发的模板。

## 上手步骤（本地 stdio 为例）

1. 选一个现成 Server，比如官方的 filesystem、fetch，或社区的 sqlite/github Server；
2. 在宿主配置里注册启动命令和参数，stdio 传输下宿主会把它作为子进程拉起；
3. 启动后确认 Client 拿到了 `tools/list`，检查工具描述和参数 schema 是否符合预期；
4. 用一个真实任务跑通 `tools/call`，观察入参出参；
5. 需要暴露私有数据时，再考虑自己写一个薄 Server，通常百来行就够。

## 踩坑点

- **工具描述就是给模型看的 prompt。** 描述含糊，模型就会选错工具、传错参。把描述写清楚，收益比换模型还大。
- **注册太多工具会污染上下文。** 每个工具的 schema 都占 token，挂到二三十个后选择准确率明显下降。按场景拆小 Server、按需挂载。
- **安全边界想清楚再开。** Server 跑在你的权限下，filesystem Server 给到根目录等于给模型整个磁盘；工具返回值也可能携带注入内容，别把外部输出直接当可信指令。
- **stdio 和远程 HTTP 是两回事。** 前者是本地子进程，后者要自己处理鉴权、超时和长连接，运维成本完全不同。
- **协议版本在演进。** 不同客户端支持的特性集合有差异，遇到"Server 正常但工具不出现"，先查双方的协议版本。

## 可复用建议

- 优先复用社区现成 Server，只对真正私有的部分自己动手；
- 业务逻辑放 Server 里而不是系统 prompt 里，工具保持单一职责、参数明确；
- 给工具输出加长度上限和结构化的错误返回，模型容错会好很多；
- 团队内维护一份 Server 清单，注明用途、权限范围和负责人。

## 总结

MCP 没有发明新智能，它做的是一件朴素的事：给"Agent ↔ 工具/数据"定义了一层通用接口，把集成成本从 M×N 压到 M+N。对做自动化和插件的人来说，价值在于可组合——今天接的 Server，明天换宿主还能用。建议先把一两个官方 Server 跑通，再决定要不要自己写，这个顺序大概率是对的。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/3dda363f53b54cba.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/1e1305645c9ed684.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/31bc2e231bb921fd.png)

