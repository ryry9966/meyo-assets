---
title: MCP 协议入门：一次看懂 Model Context Protocol 到底解决了什么问题
feedId: 32189
source: 综合讨论
publishedAt: 2026-08-09
---

## 背景：LLM 工具调用的碎片化现状

在 OpenClaw 这类 Agent 框架里，让大模型调用外部工具已经不是新鲜事。多数方案依赖自定义插件，或者直接通过函数调用（Function Calling）把工具描述塞进提示词（Prompt）。实践中会碰到几个痛点：

- 每接入一个新工具，就要写一套描述、定义 JSON Schema、处理鉴权和上下文传递。
- 工具返回的结果通常是原始 JSON 或文本，模型难以直接理解结构化上下文。
- 不同项目、不同语言的工具定义方式千差万别，复用成本高，尤其在多 Agent 协作场景下，工具接口几乎无法互通。

这些问题的本质是：**大模型与外部世界之间缺少一套通用的上下文交互协议**。

2024 年底 Anthropic 开源的 Model Context Protocol（MCP）正是冲着这个问题来的。它试图标准化 LLM 应用接入外部数据源和工具的方式，类似“工具调用的 HTTP”。

## 它解决的核心问题

MCP 官方定义直白：**一个开放协议，用于在应用和外部工具/数据源之间建立安全的、双向的上下文通道**。落到工程上，主要解决三件事：

1. **统一的工具描述与发现机制**  
   以前你需要为每个工具写 JSON Schema，放到 system prompt 或 functions 数组里。MCP 通过 `tools/list` 让客户端动态获取所有可用工具，工具自身携带描述、参数定义，模型在推理时无需硬编码。

2. **标准化的上下文传递**  
   工具调用返回的不再是 raw data，而是“资源”（Resources）或“提示模板”（Prompts），带有 MIME type、元数据等。模型可以在上下文中更自然地消费这些信息，比如把 SQL 查询结果直接当作表格理解。

3. **客户端-服务器架构解耦**  
   MCP 将“工具提供方”抽象为 MCP Server，Agent 作为 MCP Client，双方通过标准传输（目前支持 stdio、HTTP+SSE）通信。你可以用 Python 写一个数据库查询 Server，用 TypeScript 写一个文件系统 Server，而 OpenClaw 只需作为 Client 连接它们，甚至动态切换。

## 实践：让 OpenClaw 接入一个 MCP 文件系统工具

以 OpenClaw 框架为例，假设我们要让 Agent 具备本地文件读写能力（仅用于沙盒环境）。

### 1. 启动一个 MCP Server

使用 Anthropic 官方提供的示例 filesystem server：

```bash
npx @anthropic-ai/mcp-server-filesystem /tmp/sandbox
```

该命令会启动一个 MCP Server，允许安全地操作 `/tmp/sandbox` 目录下的文件。默认通过 stdio 传输，这正是 MCP 最简单的部署方式。

### 2. 在 OpenClaw 中配置 MCP 客户端

OpenClaw 内置了 MCP 客户端支持，只需在插件配置中声明连接：

```yaml
# openclaw.yaml
mcp_servers:
  - name: filesystem
    command: npx
    args: ["@anthropic-ai/mcp-server-filesystem", "/tmp/sandbox"]
```

框架会在启动时自动拉起 Server 进程，并通过 stdin/stdout 建立 MCP 会话。

### 3. 工具发现与调用

Agent 无需预先知道有哪些工具。当需要处理用户指令“列出 sandbox 下所有 .txt 文件”时，发生以下流程：

- OpenClaw 调用 `tools/list` 获取当前 MCP Server 提供的工具列表，包括 `list_directory`、`read_file` 等。
- 模型根据工具描述自主选择 `list_directory`，并生成调用参数 `{"path": "/tmp/sandbox"}`。
- MCP Server 执行并返回资源列表，每个文件带上 `text/plain` 类型和内容 URI。
- OpenClaw 将返回结果序列化后插入对话上下文，模型据此生成最终回答。

整个过程无需在 Prompt 里手写文件操作指令，也不需要对工具做二次封装。

## 踩坑点与建议

在实际接入过程中，有几个容易踩的坑：

**1. 传输层选择不当**  
Stdio 最简单，适合本地单进程管理。但若 MCP Server 需要跨网络访问或多实例复用，建议切换到 HTTP+SSE 传输。注意当前 MCP 的 HTTP 实现要求 Server 支持 Server-Sent Events，且 Client 需处理长连接断线重连。

**2. 工具权限粒度粗**  
MCP 目前对工具的权限控制比较基础，一般只到 Server 级别。如果你的文件系统 Server 既能读又能写，Agent 就可能越权删除文件。建议对重要操作拆分 Server，比如“只读文件浏览 Server”和“写入 Server”，或者在外层加一个鉴权代理。

**3. 资源过大导致上下文溢出**  
当 `read_file` 读取一个 200KB 的日志时，直接塞进上下文可能超出模型窗口。推荐在 MCP Server 端做截断或摘要，返回一个 `text/plain` 资源的同时，通过元数据标注 `truncated: true`，客户端据此提示模型“结果不完整”。

**4. 版本兼容与工具描述**  
不同 MCP Server 实现可能对规范的理解有微妙差异，比如参数类型声明是否严格符合 JSON Schema。接入前最好用官方的 `mcp validate` 工具检查 Server 描述，避免因描述错误导致模型选错工具。

## 可复用工程建议

- **搭建内部 MCP 注册中心**：为团队常用的数据源（数据库、对象存储、API 网关）封装标准化 MCP Server，让所有 Agent 项目直接复用，不再重复写 Function Calling 适配。
- **结合 OpenClaw 的多 Agent 编排**：不同子 Agent 可以连接不同的 MCP Server，主 Agent 通过工具名称前缀（如 `filesystem.list_directory`）分发调用，天然支持权限隔离。
- **监控与回放**：MCP 的调用是结构化消息流，可以在 Client 端记录所有 `tools/call` 和对应响应，方便调试和回归测试。

## 总结

MCP 解决的并不是“让模型调用工具”这个能力本身，而是**让工具调用从一次性对接变成可组合、可跨项目复用的标准化工程实践**。对于 OpenClaw 社区的用户来说，MCP 意味着我们可以把更多精力放在业务逻辑上，而不是反复为同一个数据库查询工具写不同的接入代码。

目前协议还在快速演进（已进入 2025 版草案），但核心设计已经足够稳定，值得在内部工具链中铺开试点。

---

