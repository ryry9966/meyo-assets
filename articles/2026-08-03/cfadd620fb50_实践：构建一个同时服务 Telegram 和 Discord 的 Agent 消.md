---
title: 实践：构建一个同时服务 Telegram 和 Discord 的 Agent 消息路由
feedId: 31409
source: 综合讨论
publishedAt: 2026-08-03
---

## 背景

当你把一个基于 LLM 的 Agent 接入聊天平台，最常见的起点是单平台——通常是 Telegram，因为它的 Bot API 开发体验友好。随着团队或用户群的需求扩展，很快就会出现“能不能也接入 Discord？”的问题。

两个平台的消息格式、交互模式、速率限制、身份体系完全不同，如果直接复制粘贴代码再改一版，后续维护会变成噩梦。更合理的方式是：**一个 Agent 核心，统一的消息路由层，多个平台适配器**。

这篇文章记录了我将同一个 Agent 同时接入 Telegram 和 Discord 的实践过程，重点在消息路由部分，不涉及具体 Agent 内部逻辑。适合已经在 OpenClaw、MCP、插件自动化等方向有基础，并希望扩展多平台服务能力的读者。

## 问题拆解

两个核心问题需要解决：

1. **消息归一化**：Telegram 和 Discord 的消息结构、用户标识、会话 ID、富文本格式都不一样。Agent 不能直接消费平台原生消息，需要转换成统一的内部消息对象。
2. **响应分发**：Agent 返回的回复可能是纯文本、Markdown 或结构化数据（如按钮），需要按目标平台的格式重新组装并发送回去。同时要考虑分页、回退策略、错误重试。

如果不做抽象，代码里会充满 `if tg { ... } else if discord { ... }`，随着平台增加复杂度指数级上升。

## 设计思路

整体架构分为三层：

- **Adapter 层**：负责接收 Webhook，验证签名，解析为 `IncomingMessage`，同时将 `OutgoingMessage` 渲染为平台特定格式并发送。
- **Router 层**：接收标准化消息，维持会话上下文（如 session memory 的 key），调用 Agent，拿到回复后交给 Adapter 发送。
- **Agent 层**：无状态的推理核心，输入用户消息 + 历史，输出回复文本或指令。

示意图：

```
Telegram Webhook ──► Telegram Adapter ──┐
                                       ▼
Discord Interaction ◄── Discord Adapter ── Router ──► Agent
                                       ▲
                           (IncomingMessage / OutgoingMessage)
```

消息的标准化结构设计得很简单：

```python
@dataclass
class IncomingMessage:
    platform: str          # "telegram" | "discord"
    user_id: str           # platform-native ID
    session_id: str        # 用于会话记忆的 key，如 chat_id
    text: str              # 纯文本内容
    raw: dict              # 保留原始消息，用于特殊处理
```

这样 Agent 完全不知道消息来自哪个平台，只处理 `text` 和 `session_id`。

## 具体实现步骤

### 1. 平台端配置

**Telegram Webhook**  
通过 `setWebhook` 把 bot 的更新推送到你的服务器 URL。注意国内服务器可能需要代理访问 Telegram API，建议直接用 Cloudflare Worker 作反向代理，或在同一区域部署中转。

**Discord Interactions Endpoint**  
在 Discord Developer Portal 里为 Application 配置 Interactions Endpoint URL，由 Discord 发送 `INTERACTION_CREATE` 事件。必须正确实现签名验证，否则接口会一直报 "invalid signature"。

### 2. Adapter 实现要点

**签名验证**  
Discord 使用 Ed25519 公钥签名，需要在请求处理前校验 `X-Signature-Ed25519` 和 `X-Signature-Timestamp`，否则返回 401。Telegram 的 Webhook 可通过自定义 Header 携带 secret token，在 Adapter 内比对。

**消息解析**  
Telegram 的消息中包含 `chat.id`、`from.id`、`text`（可能还有 `caption`）。Discord 的 Application Command 或 Message Component 交互中，用户消息可能在 `data.options` 或 `data.components` 里，需要扁平化为文本。

**发送回复**  
Telegram 使用 `sendMessage` API，支持 MarkdownV2 和 HTML。Discord 可用 `createInteractionResponse` 或 `editOriginalResponse`（若在 3 秒内未响应交互，需主动 followup）。这里有一个坑：Discord 交互必须在 3 秒内返回初始 ACK（type=5），否则会被标记失败。我采取的做法是：收到交互后立即返回 `type=5`，然后通过 Webhook URL 后续发送实际回复。

### 3. Router 与 Agent 集成

Router 收到 ``IncomingMessage`` 后：

1. 用 `session_id` 从缓存中拉取对话历史。
2. 组装 Prompt（system + 历史 + 新消息）。
3. 调用 Agent（LLM 或本地推理）。
4. 将 Agent 回复包装成 `OutgoingMessage`，包含 `text` 和可选的 `actions`（按钮等）。
5. 交给对应 Adapter 的 `send()` 方法。

对于同时在线的大量会话，Router 需要具备并发处理能力。我使用异步队列 + worker pool，避免一个慢回复阻塞其他用户。

## 踩坑记录

### 1. 消息长度与分片
Telegram 单条消息限制 4096 字符，Discord 为 2000。Agent 产生的长文本必须在 Router 层做智能切分，同时保证 Markdown 语法不断裂。我写了一个简单的 split 函数，优先在换行符处分段，封装为多条 `OutgoingMessage`，由 Adapter 顺序发送。Telegram 用户有时会收到 3-4 条连续消息，这在 Discord 的 embed 模式下就很难看，需要权衡是否改用文件或摘要。

### 2. 速率限制与重试
Telegram 的 API 限制约 30 msg/s 每聊天，Discord 全局限制 50 req/s。频繁调用会导致 429 Too Many Requests，必须实现指数退避重试。注意 Telegram 的 `retry_after` 参数有时会返回非常大的值（如 30s），不要傻等，可以记录告警并暂时降级。

### 3. 媒体与按钮支持
Agent 可能返回一个操作按钮或建议，但在 Discord 上按钮需要预先定义 Component，而 Telegram 用 InlineKeyboardMarkup。如果在 OutgoingMessage 里只传原始数据，每个 Adapter 需要自行转换。建议在 Adapter 中定义一套跨平台动作映射（如 `url_button`, `callback_button`），实在不支持的平台就降级为纯文本链接。

### 4. 用户状态与多轮对话
Discord 的交互式命令没有天然“会话 ID”，需要自己维护（如 DM channel ID 或 `guild_id+user_id`）。Telegram 的 `chat_id` 就天然适合。统一 session_id 的设计可以隔离不同聊天的上下文。

## 可复用建议

- **先把消息和响应抽象好**，再开始写任何一个平台的 Adapter。否则后面重构成本很高。
- **用中间件处理通用逻辑**：签名验证、请求日志、限流、异常捕获，都在 Adapter 的前置中间件完成。
- **保持 Agent 无状态**：所有对话历史由 Router 管理，Agent 只负责推理，方便随时替换模型或做 A/B 测试。
- **监控平台差异**：对每个平台的发送成功率、延迟、重试次数做独立指标，便于发现单一平台问题。
- **考虑统一 Webhook 入口**：可以在网关层（如 Nginx 或 Cloudflare Workers）根据 URL path 或 Header 分流到不同的 Adapter，减少公网暴露端口。

## 总结

让一个 Agent 同时服务 Telegram 和 Discord 并不是简单地把请求转发过去，核心在于消息归一化、响应适配以及平台特性处理。遵循 Adapter-Router-Agent 的清晰分层，可以让代码保持可维护性，日后接入 Slack、WhatsApp 等新平台时只需新增一个 Adapter。

多平台消息路由本质上是一个“协议转换”问题。把握好抽象层次，不偷懒把平台逻辑写进 Agent 里，就能保持系统的长期健康。

---

