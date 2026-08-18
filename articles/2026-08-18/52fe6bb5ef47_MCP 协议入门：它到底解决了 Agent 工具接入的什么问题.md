---
title: MCP 协议入门：它到底解决了 Agent 工具接入的什么问题
feedId: 33694
source: 综合讨论
publishedAt: 2026-08-18
---

在 OpenClaw 这类 Agent 工程里，真正拖慢进度的往往不是模型能力，而是工具接入成本。接浏览器要写 Playwright adapter，接数据库要自己封装查询函数，接文件系统要处理路径与权限，每个插件都有不同约定，最后上下文里塞满各种工具描述，模型还容易选错工具。MCP（Model Context Protocol）想解决的首要问题，就是这个“标准化”。

## 背景：工具接入的碎片化

没有 MCP 之前，常见的做法有两种：

1. **直接 Function Calling**：每个工具手写 JSON Schema，然后在 Agent 代码里写分发逻辑。缺点是每个工具都要单独维护，换一个 Agent 框架可能就要重写。
2. **插件式封装**：把工具封装成插件，但插件之间协议不统一，A 项目的浏览器插件很难直接给 B 项目的 Agent 使用。

MCP 的价值在于把 Agent 与外部能力拆成三层：

- **Host**：OpenClaw、Claude Desktop 这类宿主应用；
- **Client**：负责与 MCP Server 通信；
- **Server**：对外暴露 `Resources`（数据/资源）、`Tools`（可执行动作）、`Prompts`（提示模板）。

通信基于 JSON-RPC，传输层可以是 stdio 或 HTTP/SSE。对 Agent 来说，接入一个 MCP Server，就相当于一次性获得该 Server 下的多个工具，不需要为每个工具写胶水代码。

## 一个最小实践步骤

以文件系统 MCP Server 为例，先跑通最小闭环。

**1. 启动 Server**

```bash
npx -y @modelcontextprotocol/server-filesystem /tmp/agent-workspace
```

**2. 在 OpenClaw/Agent 中声明**

```yaml
mcp_servers:
  fs:
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-filesystem", "/tmp/agent-workspace"]
```

**3. 客户端调用流程**

连接后依次执行：`initialize` → `tools/list` 获取工具列表 → 模型需要时调用 `tools/call`。

Server 端核心不是写模型推理，而是把工具描述和 schema 写清楚：

```python
@server.list_tools()
async def list_tools():
    return [
        Tool(
            name="read_file",
            description="Read the content of a file under the allowed workspace. Use it when you need to inspect a file.",
            inputSchema={
                "type": "object",
                "properties": {
                    "path": {"type": "string", "description": "Relative path from workspace root"}
                },
                "required": ["path"]
            }
        )
    ]
```

## 实际联调中的踩坑点

1. **stdio 通道污染**  
   MCP Server 大量写 stdout 日志会污染 JSON-RPC 通道，导致客户端解析失败。日志必须走 stderr 或文件。

2. **启动生命周期卡死**  
   Server 启动慢或启动失败，会让 Agent 初始化卡住。尤其是用 `npx` 拉包，网络不好时可能几十秒才起来。建议本地安装固定版本，并给启动过程加超时。

3. **工具描述太弱**  
   模型靠 description 选择工具，写“do something”会乱调。描述要写清使用场景、参数含义、返回内容、权限边界。

4. **返回内容过大**  
   `read_file` 直接返回整个日志文件，容易撑爆上下文窗口。应该在 Server 端截断、分页或返回摘要。

5. **权限边界模糊**  
   文件 Server 如果根目录给到 `/`，等于把整机文件访问权交给模型。限定工作目录，写操作做 allowlist 或只读。

6. **Schema 不严格**  
   参数 schema 缺 `required`、缺类型限制，模型容易传乱参数。严格定义 `required`、`enum`、`maxLength`。

## 可复用建议

- 先接一个最小 Server，用 MCP Inspector 调试，确认 `tools/list` 和 `tools/call` 正常后再接入 Agent。
- 工具返回控制在 1–2k token 内，大结果落盘，返回文件路径。
- 日志集中到 stderr，避免干扰 stdio 通道。
- 记录工具调用日志，保留最近 N 条，便于排查模型误调。
- 不要把 MCP 当插件市场批量安装，每个 Server 都是潜在权限边界，按需启用。

## 总结

MCP 主要解决的是“接入标准化”和“工具描述统一”的问题，它不替代模型推理，也不自动保证安全。把它当作 Agent 工具接入的 USB-C：先跑通一个，再谈扩展。不用把它当成万能胶。

---

