---
title: OpenClaw 与外部 API 的安全握手：基于 MCP 的可靠对接实践
feedId: 32684
source: 综合讨论
publishedAt: 2026-08-12
---

## 背景

给 Agent 接上外部服务，是让它从“会聊天的模型”变成“能干活的助理”的关键一步。但在工程落地时，API 调用这件事远没有演示视频里那么丝滑——认证错误、超时无响应、返回体膨胀、工具描述不当导致 Agent 反复重试，都是常态。

OpenClaw 内置了对 MCP (Model Context Protocol) 的原生支持，让我们可以用标准化的方式把任何 HTTP API 暴露为 Agent 可发现的工具。本文将聚焦“如何稳定地完成这次握手”，而不是仅仅跑通一个 demo。

## 问题拆解

当 Agent 通过工具调用外部 API 时，我们会遇到几类典型的“断连”问题：

- **契约不清晰**：工具的参数、返回值描述不够精确，Agent 会发明参数名或错误理解返回字段。
- **超时与阻塞**：Agent 调用工具后会同步等待结果，外部 API 如果 10 秒不返回，整个任务卡死。
- **错误传播**：API 返回 500 或限流，工具直接把 HTML 错误页喂给 Agent，导致后续推理混乱。
- **状态膨胀**：某些 API 返回几百 KB 的 JSON，Agent 上下文窗口被挤爆，丢失前面的推理内容。
- **安全凭据泄露**：把 API Key 写在工具描述或日志里常见的低级失误。

## 做法：用 MCP Server 封装外部 API

我们以“天气查询 + GitHub Issue 创建”为例，展示一个可复现的对接流程。核心思路是：**每个外部服务对应一个独立的 MCP server 进程，由 OpenClaw 进程启动并管理其生命周期**。

### 1. 开发 MCP Server

采用 `mcp` Python SDK，创建一个名为 `tools_server` 的包。下面是一个只暴露天气查询工具的极简实现（真实场景按同样模式增加 Issue 创建工具）：

```python
# my_tools/server.py
import os
import httpx
from mcp.server import Server
from mcp.server.models import Tool

server = Server("external-tools")

@server.tool()
async def get_weather(city: str) -> str:
    """Get current temperature and condition for a given city.
    Returns a short string like 'Beijing: 2°C, Overcast' or an error message."""
    api_key = os.getenv("WEATHER_API_KEY")
    try:
        async with httpx.AsyncClient() as client:
            resp = await client.get(
                "https://api.weatherapi.com/v1/current.json",
                params={"key": api_key, "q": city},
                timeout=8.0
            )
            resp.raise_for_status()
            data = resp.json()
            current = data["current"]
            return f"{data['location']['name']}: {current['temp_c']}°C, {current['condition']['text']}"
    except httpx.HTTPError as e:
        return f"Error fetching weather: {e}"
    except Exception as e:
        return f"Unexpected error: {e}"

if __name__ == "__main__":
    server.run()
```

要点：

- 工具描述（docstring）明确了输入输出格式，对于 Agent 理解返回值至关重要。
- 所有异常都被截获并转为可读字符串，**绝不让调用栈进入 Agent 上下文**。
- 超时时间（8 秒）略低于 Agent 自身的工具等待超时，避免死锁。

### 2. 在 OpenClaw 中注册工具

编辑 `agent.yaml` 或对应的配置文件，增加 `mcp_servers` 段：

```yaml
mcp_servers:
  - name: external-tools
    command: python
    args: ["-m", "my_tools.server"]
    env:
      WEATHER_API_KEY: ${WEATHER_API_KEY}
```

启动 OpenClaw 时，它会自动执行这条命令，与 `external-tools` 进程建立 JSON-RPC 连接，并将 `get_weather` 等工具注入到当前 Agent 的工具列表中。无需额外手动导入。

### 3. Agent 任务的编排

下面是一个自动化例子（任务描述，非代码）：
> “每天早上 8 点检查北京天气，如果温度低于 0°C，在 GitHub 仓库 infra-alerts 创建一个标题为‘防冻提醒’的 Issue，内容包含当天温度和天气状况。”

Agent 自主规划步骤时会依次调用 `get_weather('Beijing')`，然后判断返回字符串里的温度数值，如果满足条件再调用 GitHub API 工具创建 Issue。

## 踩坑实录

- **工具描述语言不一致**：初期用中文写工具描述，Agent 会混淆英文的 JSON 字段名。后来统一工具描述、参数名、错误信息全部英文，稳定性明显提升。
- **返回体未裁剪**：某次内部 API 返回了 200KB 的日志 JSON，Agent 上下文直接溢出，任务静默失败。后续工具返回时加上了摘要裁剪逻辑：若结果超过 4KB，只保留前 2KB 和尾部摘要。
- **网络抖动导致级联重试**：Agent 发现超时后会立即重试同一个工具，API 侧因限流再报错，形成死循环。我们在工具层加入了简单的退避策略：连续相同参数的调用只允许每 30 秒 1 次，其余返回缓存或“请稍后重试”。
- **环境变量注入失效**：OpenClaw 配置里使用 `${VAR}` 语法，但启动时 shell 未展开，工具进程拿不到 API Key。最终确保 OpenClaw 进程本身运行在一个已 export 环境变量的 shell 中，或改用 `env` 直接指定明文（仅开发环境）。

## 可复用的工程建议

- **将 OpenAPI spec 转换为 MCP 工具**：如果你的服务已经有一套 OpenAPI 文档，可以用 `openapi-to-mcp` 这类工具自动生成 MCP server，避免手工维护工具定义。确保转换后的 tool description 包含返回示例，帮助 Agent 做推理。
- **统一错误结构**：所有工具在异常路径下都返回 `{"error": "reason", "detail": "..."}` 这样的 JSON 字符串，Agent 能轻易判断“调用成功”还是“需要降级”。
- **可观测性**：在 MCP server 中对每个工具调用打印一条结构化日志，包含调用时间、输入参数、耗时、返回长度和是否成功。在 OpenClaw 的任务回放中能快速定位哪些步骤卡在外部依赖。
- **特权分离**：外部 API 的认证凭据只存在于 MCP server 进程的环境变量中，Agent 内永远看不到原始 Key。每次新增服务时提醒自己：**工具应像操作系统的 syscall，数据进入 Agent 前必须脱敏或裁剪**。

## 总结

Agent 与外部 API 的对接不只是简单的 HTTP 调用，而是在异步、易出错的网络边界上建立一份可靠的契约。基于 MCP 的对接方式，将 API 包装成带有严格描述、超时、错误处理和输出限制的工具，是让自动化任务稳定跑在几十小时时间跨度上的关键。这次“握手”做得越扎实，后续的任务编排才会越省心。

---

