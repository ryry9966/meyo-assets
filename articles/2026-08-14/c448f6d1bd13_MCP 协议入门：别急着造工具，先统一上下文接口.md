---
title: MCP 协议入门：别急着造工具，先统一上下文接口
feedId: 33099
source: 综合讨论
publishedAt: 2026-08-14
---

# MCP 协议入门：别急着造工具，先统一上下文接口

## 背景：Agent 工具接入的 M×N 问题

在 OpenClaw、Claude Desktop、VS Code 这类宿主环境里做 Agent 自动化时，最容易卡住的往往不是模型能力，而是外部工具接入。文件系统、浏览器、数据库、GitHub、内部 API，每接一个工具就可能要按宿主平台重写一遍。更麻烦的是，每个宿主对工具描述、鉴权、超时、重启、上下文注入都有自己的一套。

MCP（Model Context Protocol）想解决的正是这个碎片化问题。它把“模型如何发现工具、读取资源、使用提示模板”抽象成统一协议，让一个 MCP Server 可以被不同客户端复用。

## 问题：不只是“多写一个插件”

实际工程里，传统做法有几个明显痛点：

- **适配成本高**：N 个工具 × M 个宿主，很容易变成 N×M 个插件。
- **上下文混乱**：工具输出直接塞进 system prompt，格式不统一，token 消耗不可控。
- **权限与生命周期散落**：每个插件自己做权限、超时、重启，行为不一致。
- **可移植性差**：给 A 平台写的插件，在 B 平台基本不能复用。

MCP 的价值不在于提升模型智商，而在于把工具接入变成协议层的事。

## 做法：先跑通一个最小 MCP 接入

MCP 有三个核心角色：

- **Host**：OpenClaw、Claude Desktop 等宿主程序。
- **Client**：宿主内的 MCP 客户端，负责与 Server 通信。
- **Server**：暴露 tools、resources、prompts 的进程。

常见的 transport 是 stdio 和 HTTP SSE。入门建议先用 stdio，最简单。

以 OpenClaw 接入文件系统 MCP Server 为例：

1. 确认本机有 Node.js 环境。
2. 选择一个社区 Server，例如 `@modelcontextprotocol/server-filesystem`。
3. 在 OpenClaw 的 MCP 配置里加入：

```json
{
  "mcpServers": {
    "fs": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/allowed"]
    }
  }
}
```

4. 重启客户端，查看工具列表是否出现 `read_file`、`write_file`、`list_directory` 等。
5. 让 Agent 实际读取一个白名单目录下的文件，观察它是否能按参数 schema 正确调用。

跑通这一步，比直接自建 Server 更有意义。你会看到 MCP 的边界在哪里：它负责工具发现和调用，但不负责模型判断。

## 踩坑点

- **npx 下载慢或失败**：国内网络下 `npx -y` 可能卡住。建议先全局安装包，再直接配置可执行文件名，例如 `npm install -g @modelcontextprotocol/server-filesystem`，command 用 `mcp-server-filesystem`。
- **路径空格与转义**：args 里的路径如果带空格，JSON 中要确保是独立字符串，不要拼进 `npx` 的同一段命令里。
- **stdio 服务崩溃难排查**：MCP Server 一旦异常退出，客户端可能只表现为工具不可用。建议先用 `@modelcontextprotocol/inspector` 手动发 JSON-RPC 请求，验证 `initialize`、`tools/list`、`tools/call`。
- **工具描述太模糊**：模型调用错误十有八九是 schema 和 description 写得不清。描述要写明“何时用、输入是什么、输出是什么、限制是什么”。
- **上下文膨胀**：如果 Server 把整个目录树或数据库全量导出，token 会迅速失控。务必在 Server 侧限制资源大小，或客户端做裁剪。
- **客户端能力差异**：有些客户端只支持 tools，不支持完整的 resources/prompts。接入前先验证 OpenClaw 当前版本的能力范围。

## 可复用建议

1. **先复用，后定制**：社区已经有不少 filesystem、fetch、github、database Server，先确认现有工具是否够用。
2. **自定义 Server 从最小工具开始**：用 Python 或 TS SDK 写一个只读 tool，先验证 stdio 通信，再扩展。
3. **权限最小化**：目录白名单、只读 token、环境变量 secret，不要图省事给全局权限。
4. **把描述当 prompt 的一部分**：工具描述、参数说明、错误提示都会被模型读到，值得花时间打磨。
5. **纳入版本管理**：MCP Server 的配置、启动参数、环境变量模板应放进 Git，避免手改服务器。
6. **设置超时与重启策略**：在 OpenClaw 中给 MCP Server 配置合理的 timeout 和 restart，避免一个挂掉的 Server 拖死整个 Agent 流程。

## 总结

MCP 解决的是上下文接入的工程化问题，不是模型推理问题。它让工具可移植、可测试、可下线，减少重复适配。对 OpenClaw 用户来说，协议统一之后，真正值得投入的是工具设计、权限策略和上下文预算，而不是继续写第 N 个平台适配层。

---

