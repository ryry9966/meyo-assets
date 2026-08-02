---
title: 一个 Agent 两个平台：在 OpenClaw 中为 Telegram 与 Discord 构建统一消息路由
feedId: 31313
source: 综合讨论
publishedAt: 2026-08-02
---

## 背景

如果你正在用 OpenClaw 这套 Agent 框架搭建内部助手，迟早会碰到一个现实需求：同一个 Agent 需要同时接入多个即时通讯平台。最常见的组合是 **Telegram + Discord** —— 前者轻量、API 友好，后者在社区运营中几乎无法绕开。

直接为每个平台单独跑一个 Agent 实例当然可行，但会带来几个问题：

* 上下文和记忆割裂，无法做到用户跨平台的连贯体验；
* 每一次逻辑变更都要同步修改多个实例，不利于维护；
* 不同平台的回复格式、速率限制各不相同，手工适配容易漏掉边界情况。

这篇文章会从工程角度介绍一种相对干净的方案：**在 OpenClaw 里用 MCP 插件分别对接 Telegram 和 Discord，由同一个 Agent 实现多平台消息路由**。重点放在可复用的设计思路与趟坑记录上，不涉及特定云服务或商业产品。

## 问题拆解

统一路由要解决的几个核心问题：

1. **消息汇聚**：两个平台的消息能以统一的结构进入 Agent 的处理流程。
2. **用户与会话标识**：Telegram 用 `chat_id`，Discord 用 `channel_id` + `guild_id`（服务器）和 `user_id`，需要映射成一致的内部标识，同时支持会话隔离。
3. **回复分发**：Agent 生成响应后，能准确找到消息来源对应的平台及会话，并以该平台能接受的格式发出。
4. **工程可靠性**：异步消息处理不能互相阻塞、平台 API 的异常要有缓冲和重试、避免因 Agent 出错导致某一边彻底失联。

## 做法与步骤

### 1. 准备两边的 Bot 与 MCP 插件

首先在 Telegram 通过 [@BotFather](https://t.me/BotFather) 创建 Bot，拿到 `token`；在 Discord 开发者后台创建 Application，生成 Bot 并获取 `token`，同时勾选必要的 Gateway Intents（`GUILD_MESSAGES`、`MESSAGE_CONTENT` 等）。

OpenClaw 当前生态中，已经存在社区维护的 `mcp-telegram` 和 `mcp-discord` 插件，它们分别封装了对应的 Bot API。在 OpenClaw 的 agent 配置里，将它们以 MCP 服务的方式挂载进来，大致结构如下：

```yaml
mcp_servers:
  - name: telegram-in
    plugin: mcp-telegram
    config:
      token: ${TG_BOT_TOKEN}
      mode: polling   # 早期建议用 polling，避免 webhook 公网暴露
  - name: discord-in
    plugin: mcp-discord
    config:
      token: ${DISCORD_BOT_TOKEN}
```

> 注意：这里刻意区分了 `telegram-in` 与 `discord-in`，而不是直接用一个叫 `chat` 的混合插件，目的是保持每个通道的异常隔离——想象一下 Discord 连接断开不会影响 Telegram 收消息。

### 2. 标准化消息格式

插件接收到的原始消息结构差异很大。我们需要在 Agent 的预处理器中，把它们转成统一的内部格式：

```text
{
  "platform": "telegram" | "discord",
  "session_id": "<platform-user-or-channel-id>",
  "user_id": "<user_unique_id>",
  "content": "<text>",
  "platform_meta": { ... }   # 保留原始元数据用来回复
}
```

**session_id 生成策略**：Telegram 私聊直接用 `chat_id`，群聊用 `chat_id` 拼上 `message_thread_id`；Discord 私信用 `channel_id`（DM 的频道 ID），服务器频道则用 `guild_id + channel_id` 拼接，必要时再加入 `thread_id`。这样做的好处是，同一个群或频道内的多人对话可以维持在同一个会话上下文中，和用户的预期行为一致。

内部 `user_id` 用 `platform + ":" + original_user_id` 的形式，比如 `telegram:123456` 和 `discord:789012`，Agent 可以借此识别跨平台的同一自然人（如果未来接入统一身份）。

### 3. Agent 处理与路由回复

OpenClaw 中的 Agent 本质上是一个消息处理管道。在路由层，我们用同一个 `handler` 接收来自 `telegram-in` 和 `discord-in` 的标准化消息，然后调用 LLM、执行工具、生成回复。

回复阶段需要“反向路由”：根据消息的 `platform` 和 `platform_meta` 决定调用哪一个 MCP 的输出接口。伪代码示意：

```python
def route_response(message, response_text):
    if message.platform == "telegram":
        tg = mcp.get_server("telegram-in")
        tg.send_message(
            chat_id=message.platform_meta.chat_id,
            text=response_text,
            parse_mode="MarkdownV2"
        )
    elif message.platform == "discord":
        dc = mcp.get_server("discord-in")
        dc.send_message(
            channel_id=message.platform_meta.channel_id,
            content=response_text
        )
```

这种显式的分支虽然看起来“土”，但好处是每个平台对格式、长度、embed、附件等要求的处理都高度透明，后期单独调整时不会相互污染。

### 4. 消息去重与循环防护

两个平台有时会通过第三方桥接软件（如经典的 Discord-Telegram 桥接）互通，这种情况下你的 Agent 可能同时收到来自同一对话的两份消息，形成回环。一个简单有效的办法是：**在处理前，为每条入站消息计算一个幂等键**，由 `platform + channel_id + message_id` 组成，在 TTL 缓存（如 5 分钟）中查重，已处理的直接丢弃。

另一点很容易被忽略：Agent 自身发出的回复，有可能被平台以某种方式再次传入（例如有的桥接机器人会把其他人的回复转回来）。检查消息的 `author_id` 是否为 Bot 自己，是则跳过，是更保险的做法。

## 踩坑记录

### Telegram：Markdown 转义地狱

Telegram Bot API 的 `parse_mode` 支持 MarkdownV2，但要求几乎所有标点都必须转义。Agent 吐出的文本通常带有一堆特殊字符（`*`, `_`, `~`, `>`）。如果在输出阶段不做任何处理就直接推送，消息会发送失败或者格式乱掉。

建议在 `send_message` 前加一层 **轻量转义函数**，只针对 Telegram 用 `parse_mode: MarkdownV2` 的情况转义，或者降低复杂度直接使用 `parse_mode: HTML` 并做 HTML 实体编码。个人经验是：如果 Agent 输出中不含用户输入，用 HTML 更省心。

### Discord：速率限制与 embed 尺寸

Discord 对单个频道有全局速率限制（全局 50 次/秒），而且 embed 的文字字段有 1024 字符上限。Agent 有时会输出很长的一段分析，直接塞进 embed 会截断。

解决方案：在 `discord` 分支中检查回复长度，超过 1900 字符时分片发送，或者降级为普通文本消息而不带 embed。同时在发送循环中加入一个极轻量的速率调度器（0.2s 间隔即可避免绝大多数 429）。

### 连接恢复与健康检查

无论是 polling 模式还是 WebSocket 连接，MCP 插件与平台的连接都可能意外断开。在 OpenClaw 的配置中为每个 MCP 服务开启 `auto_restart` 和 `health_check_interval`，并在 Agent 初始化时捕获初始连接错误，避免一个插件崩溃导致整个 Agent 进程退出。

## 可复用建议

1. **用环境变量管理凭据**，不硬编码 token，便于在不同开发/生产环境中切换。
2. **消息路由逻辑抽成独立的 Router 类**，输入平台消息，输出标准化内部消息，并负责反向分发。这样新增平台（如 Slack、WhatsApp）时只需实现新的适配器，核心 Agent 逻辑不动。
3. **记录一次完整的跨平台对话链路日志**：包括入站消息的标准化前/后、Agent 消费的 prompt、生成的回复、发送结果。这比单独看每个平台的日志高效得多。
4. **先从单平台稳定运行开始**，再接入第二个平台，避免一上来就面对复合故障。

## 总结

通过 OpenClaw + MCP 插件的方式，用同一个 Agent 服务 Telegram 和 Discord 是完全可行且容易工程化的。核心工作无非是**消息标准化、会话标识生成、回复分发与格式适配**，再加上一层保护性的去重与限流逻辑。

这样做之后，维护一个内部知识助手或社区管理机器人，就从一个“每平台一坨”的混乱状态，收敛到了一个集中的处理管线。后续无论是要接入新的消息渠道，还是提升 Agent 本身的推理能力，这个统一入口都会让你少背很多技术债。

---

