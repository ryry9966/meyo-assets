---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 34790
source: 综合讨论
publishedAt: 2026-08-26
---

在 Agent 开发里，最常见的需求不是“让模型更强”，而是让模型能稳定地调用一个文件、查一个数据库、发一个 HTTP 请求。MCP（Model Context Protocol）做的事情，就是把这件事从“每个 Agent 写一遍胶水代码”变成“一套协议、一个可复用的工具服务”。

## 一、背景：工具接入的 N×M 问题

过去我们给 Agent 接工具，通常有三步：在 prompt 里写 function schema；在代码里实现函数；在运行时做参数校验、序列化和错误回传。这个链路在单个工具时没毛病，但当你有 3 个 Agent、20 个工具，就变成 60 个组合。

更麻烦的是，每个工具对“上下文”的理解不一样：有的需要读取文件片段，有的需要数据库 schema，有的需要网页内容。模型侧只能靠 prompt 描述去猜，跑不通时排查成本很高。

MCP 把工具、资源和提示这三类能力抽象成统一协议：MCP server 暴露工具列表和 schema，MCP client（Agent 运行时）通过 JSON-RPC 调用。Anthropic 提出后，逐渐被 Agent 框架和 OpenClaw 这类自动化平台吸收。

## 二、它真正解决的问题

严格说，MCP 不提升模型智商，也不自动解决工具调用准确率。它解决的是：

1. **接入标准化**：工具作者只需要实现一个 MCP server，任何兼容 MCP 的 Agent 都能用。
2. **上下文结构化**：资源（文件、日志）、工具（函数）、提示模板被分开管理，不再把全部内容塞进 system prompt。
3. **进程隔离**：MCP server 独立运行，崩溃不会直接拖垮 Agent 主进程。
4. **可复用**：社区里现成的 filesystem、fetch、database server 可以直接挂载，不必重复造轮子。

## 三、一个最小接入流程

假设你在 OpenClaw 里接一个本地的 filesystem MCP server：

1. 确认运行时支持 MCP 客户端。OpenClaw 通常在配置文件中提供 `mcpServers` 类似的注册项。
2. 准备 server。以 stdio 方式启动，例如：

```json
{
  "mcpServers": {
    "fs": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/tmp/sandbox"]
    }
  }
}
```

3. 启动后看工具列表是否出现 `read_file`、`write_file` 等。注意先给 `/tmp/sandbox` 这样的只读或受限目录。
4. 在 Agent 里配置工具开关，只保留需要的工具，避免一次暴露几十个。
5. 跑通一个闭环：读取文件 → 模型理解内容 → 写入新文件 → 校验。

## 四、踩坑点

- **stdio 与 HTTP/SSE 混用**：本地 Agent 常用 stdio，远程或 Web 端可能需要 SSE/Streamable HTTP。不要把 stdio 的 MCP server 直接暴露到公网。
- **相对路径和工作目录**：MCP server 的 `args` 里相对路径基于启动它的工作目录，Agent 重启后可能变化。建议全部用绝对路径。
- **参数校验不全**：别指望模型一定按 schema 传参。MCP server 内部仍要做防御性校验，例如路径穿越、超长输入、危险命令。
- **工具爆炸**：挂载 30 个工具后，模型选择正确工具的概率会下降。按任务域分组，按需启用。
- **协议版本差异**：MCP 还在演进，客户端和 server 的初始化版本不一致时会报 JSON-RPC 错误。先用 MCP Inspector 单独验证 server。
- **权限不是魔法**：MCP 本身不提供细粒度授权。一个可写文件系统的 server 就是高危工具，建议在 OpenClaw 层做二次确认或只读模式。

## 五、可复用建议

- 先用 MCP Inspector 调试 server，确认 `initialize`、`tools/list`、`tools/call` 正常，再接 Agent。
- 把 MCP server 当成独立服务来维护：有版本号、有测试、有日志。
- 工具描述要写清楚“什么时候用、什么时候不能用”，比 schema 还重要。
- 在 OpenClaw 里给 MCP 工具加命名空间，避免和内置工具重名。
- 如果只是单个 Agent 调用单个函数，直接写一个普通函数可能比引入 MCP 更省事。MCP 的价值在跨 Agent 复用和社区生态，不在仪式感。

## 总结

MCP 解决的是 Agent 工具接入的“最后一公里”：把原来分散在 prompt、代码、配置里的工具协议抽象出来，让工具可被发现、被调用、被复用。它不是一个新框架，更像一个胶水协议。对 OpenClaw 用户来说，正确用法不是“把所有东西都变成 MCP server”，而是把高频、跨任务、需要隔离的工具沉到 MCP 层；一次性、单点工具继续留在 Agent 内部函数里。这样既能享受生态红利，又不会把系统复杂度抬到没必要的高度。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/5f2c252b86ad3442.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/765f88a6659a350c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/617dc1785387f2d7.png)

