---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 31282
source: 综合讨论
publishedAt: 2026-08-02
---

## 背景

在一个典型的自动化工作流里，同一个 Agent 经常需要向不同的渠道推送通知。比如：

- 运维监控告警，既要发到技术团队的 Telegram 群，也要同步到 Discord 频道供其他时区的同事查看。
- CI/CD 流水线结束后，Agent 把结果摘要同时推送到两个平台，避免“消息孤岛”。
- 社区机器人根据用户偏好，允许选择通过 Telegram 或 Discord 接收回复。

问题来了：**如何让一个 Agent 同时具备 Telegram 和 Discord 的发送能力，而不把通道逻辑硬编码在核心逻辑里？** 如果你的 Agent 已经跑在 OpenClaw 之上，并且正在用 MCP 扩展能力，那么答案会很自然——用 MCP Server 将不同平台的发送接口抽象成统一的工具。

---

## 思路：用 MCP 做通道适配

OpenClaw 的插件机制允许通过 MCP (Model Context Protocol) 为 Agent 挂载外部工具。我们可以编写一个轻量级的 MCP Server，内部集成 Telegram Bot API 与 Discord Bot API，暴露出两个独立工具，例如：

- `send_telegram_message(text: str, chat_id?: str)`
- `send_discord_message(content: str, channel_id?: str, embed?: object)`

Agent 只需要根据上下文调用这些工具，完全不用关心底层的 HTTP 请求、重试策略或速率限制。核心逻辑里一句 `send_telegram_message` 或 `send_discord_message` 即可完成跨平台投递。

此外，如果你希望 Agent 自主判断该发到哪里（例如根据消息的紧急程度或用户来源），可以在 system prompt 中说明路由规则，让模型决定调用哪一个工具，甚至同时调用两个。

---

## 实现步骤

### 1. 创建并配置 Bot

- **Telegram**：通过 [@BotFather](https://t.me/BotFather) 创建一个 Bot，保存 API token。将 Bot 拉入目标群组并给予发送消息权限，记录 chat_id（可通过 `getUpdates` 获取）。
- **Discord**：在 [Discord Developer Portal](https://discord.com/developers/applications) 创建 Application，开启 Bot 功能，复制 token。在 OAuth2 URL Generator 中选择 `bot` 和 `Send Messages` 权限，生成邀请链接将 Bot 加入服务器。开启 `MESSAGE CONTENT INTENT` 以便 Bot 读取消息内容（如果未来需要双向交互），但纯发送场景下仅需发送权限。

### 2. 编写 MCP Server

使用 Python 和 `mcp` 库快速实现。核心依赖：

- `python-telegram-bot`（或直接使用 `httpx` 调 Telegram API）
- `discord.py` 或 `aiohttp` 调 Discord REST API

简化版示例工具定义（用 FastMCP）：

```python
from mcp.server import FastMCP
import httpx

mcp = FastMCP("messaging-router")

TELEGRAM_TOKEN = "xxx"
DISCORD_TOKEN = "yyy"
DEFAULT_CHAT_ID = "-100..."
DEFAULT_CHANNEL_ID = "123..."

@mcp.tool()
async def send_telegram_message(text: str, chat_id: str = None):
    """Send a plain text message via Telegram"""
    target = chat_id or DEFAULT_CHAT_ID
    url = f"https://api.telegram.org/bot{TELEGRAM_TOKEN}/sendMessage"
    async with httpx.AsyncClient() as client:
        resp = await client.post(url, json={
            "chat_id": target,
            "text": text[:4096]  # Telegram max length
        })
        resp.raise_for_status()
        return resp.json()

@mcp.tool()
async def send_discord_message(content: str, channel_id: str = None, embed: dict = None):
    """Send a message to a Discord channel, optionally with an embed"""
    target = channel_id or DEFAULT_CHANNEL_ID
    url = f"https://discord.com/api/v10/channels/{target}/messages"
    headers = {"Authorization": f"Bot {DISCORD_TOKEN}"}
    payload = {"content": content[:2000]}
    if embed:
        payload["embeds"] = [embed]
    async with httpx.AsyncClient() as client:
        resp = await client.post(url, json=payload, headers=headers)
        resp.raise_for_status()
        return resp.json()
```

> 注意：这只是最小实现，生产环境需要完善异常处理、日志与限速控制。

### 3. 将 MCP Server 挂载到 OpenClaw Agent

在 OpenClaw 的插件配置中添加该 MCP Server 的启动命令（例如 `python messaging_router.py`），Agent 启动后即可在工具列表中看到 `send_telegram_message` 和 `send_discord_message`。

在提示词里明确工具用途，例如：

```
你需要将重要的运维告警同时发送到 Telegram 和 Discord。
如果用户要求静默通知，则只发 Discord；否则两个通道都发。
```

Agent 会自动在需要时调用工具，完成跨平台通知。

---

## 踩坑记录

1. **消息长度限制**  
   Telegram 单条文本上限 4096 字符，Discord 为 2000 字符。如果源内容较长，需要实现分段发送逻辑。一个稳健的做法是在工具内部做 truncation 并打上省略标记，或者返回错误让 Agent 自行截断重试。

2. **Embed 与 Markdown 差异**  
   Telegram 支持 MarkdownV2 和 HTML，但语法严格，部分字符需要转义。Discord 的 markdown 相对宽容，但嵌入 (embed) 的字段有限制（title 256 字符、field value 1024 字符）。如果希望保持格式一致，建议在 Agent 侧输出纯文本，让各平台的工具各自处理富文本模板。

3. **速率限制（Rate Limit）**  
   Telegram 官方限制一个 Bot 对同一 chat 的消息频率为每秒约 30 条，Discord 为每 5 秒 5 条。如果你的 Agent 可能批量发送通知，需要在工具内部实现简单的队列或指数退避重试，否则 HTTP 429 错误会直接抛给 Agent，可能导致任务失败。

4. **错误上报与监控**  
   消息发送失败时，Agent 应能感知失败并记录。建议在工具实现在捕获异常后返回结构化错误信息（如 `{"error": "Telegram 403 Forbidden"}`），而不是让 MCP 调用直接抛出异常。这样 Agent 可以依据错误类型决定是否重试或降级。

5. **安全与配置管理**  
   Token 和默认 channel/chat_id 不要硬编码在代码中，应通过环境变量注入。多人协作时注意 `.env` 文件的权限，避免误提交到版本控制。

---

## 可复用的工程化建议

- **抽象统一通道接口**：如果你的项目涉及更多平台（Slack、飞书等），可以将各通道的发送逻辑封装成统一的 `send_message(channel, payload)` 工具，由 Agent 选定通道，减少工具数量。
- **参数化默认目标**：通过 MCP Server 的配置参数或环境变量设定默认 chat_id 和 channel_id，Agent 在不指定目标时也能发送到预置群组。
- **日志与链路追踪**：在 MCP Server 内每次发送请求都记录日志，并带上 Agent 传入的 trace_id（可由 OpenClaw 传递），便于排查“Agent 说发了但用户没收到”的故障。
- **通道健康检查**：提供 `health_check` 工具让 Agent 在启动时验证 Telegram 和 Discord 的连接状态，提前暴露 token 失效或权限不足的问题。
- **“仅模拟”模式**：开发测试时可以增加 DRY_RUN 环境变量，此时工具不实际调用 API，而是将输出打到 log，方便验证路由逻辑。

---

## 总结

通过 MCP 将不同消息平台的 API 适配成统一的工具，你的 OpenClaw Agent 可以在不增加核心复杂度的情况下，灵活实现跨平台消息路由。关键是做好异常处理、速率限制和错误反馈，让 Agent 能可靠地完成“最后一公里”的通知投递。

这种模式还可以自然扩展：未来如果需要支持更多平台，只需新增对应的 MCP 工具，无需改动 Agent 的主体逻辑，符合工程中“开放-封闭”原则。

---

