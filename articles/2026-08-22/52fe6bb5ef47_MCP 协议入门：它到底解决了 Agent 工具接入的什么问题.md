---
title: MCP 协议入门：它到底解决了 Agent 工具接入的什么问题
feedId: 34231
source: 综合讨论
publishedAt: 2026-08-22
---

## 背景

在 Agent 开发里，工具接入长期处于“一个平台一套适配器”的状态。同一个网页抓取能力，ChatGPT 插件写一次，Claude 工具写一次，本地自动化脚本还要再封装一次。Agent 侧则要分别维护工具列表、参数描述、调用格式和错误处理。工具越多，N×M 的适配成本越高。

MCP（Model Context Protocol）想把集成层标准化：统一工具发现、参数描述、调用和上下文传递方式。它不是让模型变强，而是让工具接入变简单。

## 解决了什么问题

具体来说，MCP 主要解决三类问题：

1. **接入格式不统一**：每个平台对工具描述、schema、调用返回都有自己格式。MCP 用 JSON-RPC + JSON Schema 统一工具定义。
2. **资源/提示/工具混在一起**：以前一个“工具”可能既返回数据，又写文件，还带提示词。MCP 把 Tools、Resources、Prompts 分开建模，职责更清晰。
3. **客户端和服务端生命周期不一致**：MCP 定义了 initialize、tools/list、tools/call 等握手和调用流程，客户端可以动态发现能力，不需要预编译进代码。

## 最小可复现步骤

下面以 OpenClaw 作为 MCP client 为例，接入一个只读的“查时间”工具。

### 1. 写一个 MCP server

示例代码采用伪代码，不同 SDK 版本 API 略有差异：

```python
server = create_mcp_server("time-server")

@server.tool(
    name="get_current_time",
    description="返回当前 UTC 时间",
    input_schema={
        "type": "object",
        "properties": {
            "timezone": {
                "type": "string",
                "description": "IANA 时区，例如 Asia/Shanghai"
            }
        },
        "required": ["timezone"]
    }
)
def get_current_time(timezone: str) -> str:
    return f"timezone={timezone}"
```

通过 stdio 启动。注意：**不要用 `print()` 打日志**，stdio 通道会把它当成协议消息，导致解析失败。日志写到 `stderr`。

### 2. 在 OpenClaw 中注册

以本地 stdio server 为例，配置里增加一条 MCP server：

```json
{
  "mcpServers": {
    "time": {
      "command": "python",
      "args": ["/absolute/path/to/time_server.py"]
    }
  }
}
```

不同客户端字段略有差异，OpenClaw 通常放在插件配置或 `mcp.json` 中。

### 3. 测试调用

启动后客户端会先发 `initialize`，然后 `tools/list`。确认工具出现在可用列表里，再让 Agent 用自然语言触发“查一下上海时间”。如果参数缺失，模型会根据 schema 补全。

## 踩坑点

- **stdio 日志污染**：最高频的坑。任何 `print`、`console.log` 都会破坏协议。排查时只看 stderr。
- **JSON Schema 描述不充分**：模型依赖 `description` 理解工具。属性描述不清，或 `required` 缺失，容易出现填错参数或漏参数。
- **工具名和描述不要频繁改**：Agent 配置里可能缓存了工具列表或依赖工具名做路由。改名后要重新加载。
- **权限边界模糊**：MCP server 跑在本地时拥有当前用户权限。如果工具支持写文件或执行命令，等于给模型开放了高危操作。建议先只读，写操作加白名单。
- **协议版本不匹配**：MCP 目前仍在演进。SDK 升级后，如果 client 仍用旧版本，可能出现初始化失败或 `tools/list` 返回格式不识别。先固定版本再升级。
- **大结果不做截断**：工具一次返回几万 token，会挤占上下文窗口，甚至触发 client 抛错。加 `limit` 参数或分页。

## 可复用建议

1. **先做成只读工具**：查询类、搜索类优先，写操作放第二批。
2. **工具实现与传输分离**：核心逻辑写成纯函数，MCP 层只做参数解析和返回包装，方便单测。
3. **给每个 MCP server 加一个 `ping` 工具**：返回 `ok`，用于排障时确认服务是否活着。
4. **用 JSON Schema 严格约束**：不要用 `anyOf` 或过深嵌套，能用 `enum` 就不用自由字符串。
5. **配置模板化**：开发环境用相对路径，生产环境用绝对路径，`env` 里放密钥时注意不要提交到仓库。

## 总结

MCP 没有解决模型推理、规划或记忆的问题。它解决的是工程侧最脏最累的那部分：让工具描述、发现、调用和上下文传递有一套可复用约定。对 OpenClaw 用户来说，花半天把现有插件改造成 MCP server，长期收益是后续新 Agent、新平台接入时不用再重写一遍适配逻辑。

先把协议版本固定住，从只读工具开始，比一次性接入一堆服务更实际。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/c1924be32e952c81.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/6bf5b7520aab25a2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/dfe8ee6223c6a98d.png)

