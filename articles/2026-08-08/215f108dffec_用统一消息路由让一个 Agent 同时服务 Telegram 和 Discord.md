---
title: 用统一消息路由让一个 Agent 同时服务 Telegram 和 Discord
feedId: 32035
source: 综合讨论
publishedAt: 2026-08-08
---

## 背景

在使用 OpenClaw 搭建个人助手或自动化 agent 时，常见需求是让同一个 agent 逻辑同时处理来自 Telegram 和 Discord 的消息。例如，一个用于查询服务器状态的 agent，既希望在群聊里通过 Telegram bot 调用，也方便 Discord 社区的成员使用。又或者，一个自动翻译 bot 需要跨平台保持一致行为。

直接为每个平台各写一份接入代码并不难，但维护两套几乎相同却又有微妙差异的 handler，会很快演变成噩梦。平台消息格式、富文本语法、线程/频道模型都不相同，加上未来可能接入 Slack、Matrix 等，单一实现模式不再适用。

这篇文章记录一种工程化的实践：**通过消息路由层，将不同平台的输入抽象为统一消息结构，调用同一个 Agent 处理后再回复到原平台**。所有代码基于 Python，可直接与 OpenClaw 项目中的 agent 模块配合。

## 核心问题

要把一个 agent 实例同时接入 Telegram 与 Discord，需要解决几个问题：

1. **消息结构异构**：Telegram 使用 `Message` 对象（支持 MarkdownV2 / HTML），Discord 使用 `Message` 对象（支持 embed、stickers、组件），用户信息、频道标识、回复关系都长得不一样。
2. **会话与上下文管理**：每个平台的聊天上下文不同，需要将平台特定的对话标识映射到 agent 理解的 session ID，并保持会话存储隔离。
3. **响应路由**：Agent 产生的回复必须发回正确的平台、正确的频道/私聊，无论是文本还是带操作按钮的消息。
4. **可靠性**：Webhook 重试、速率限制、长文本分片、图片/文件处理都会在不同平台表露出不同问题。

## 做法与步骤

### 1. 定义统一的内部消息协议

创建一个 `UnifiedMessage` 数据类，只保留 agent 处理所需的最小字段：

```python
@dataclass
class UnifiedMessage:
    platform: str          # "telegram" / "discord"
    chat_id: str           # 平台内唯一会话标识
    user_id: str
    text: str
    message_id: str        # 用于去重和回复引用
    reply_to: Optional[str] = None
    attachments: List[str] = field(default_factory=list)
```

这样 agent 看到的永远是同一个结构，不需要关心平台细节。

### 2. 构建平台适配器

为每个平台实现 `PlatformAdapter` 接口，至少包含两个方法：

- `incoming(payload) -> UnifiedMessage`
- `outgoing(response, unified_msg) -> 平台实际发送动作`

例如，Telegram 适配器从 webhook payload 中提取 `chat.id`、`text`、`from.id`，把 `message_id` 作为去重键；发送时调用 `sendMessage` API，注意文本长度限制，超过 4096 字符时自动分片或用文件发送。

Discord 适配器从 Gateway 事件中构造 `UnifiedMessage`，把 `channel_id` 作为 `chat_id`。回复时使用 `channel.send()`，并处理 Discord 的 2000 字符限制，将长回复转为文件或分条发送，也可按需生成 embed。

### 3. 路由与处理流程

整体架构：**平台事件 -> 适配器 -> 统一消息 -> Agent -> 统一回复 -> 适配器 -> 平台发送**。

具体流程：

- 对于 Telegram，使用 `python-telegram-bot` 库或直接 Flask webhook 接收更新。
- 对于 Discord，使用 `discord.py` 的 `on_message` 事件。
- 两个入口都将原始消息交给各自的适配器，转换为 `UnifiedMessage`。
- 所有统一消息投入一个队列（如 `asyncio.Queue`），由一个 worker 从队列取出后调用 agent 逻辑（如 `agent.process(unified_msg)`）。
- Agent 返回一个 `UnifiedResponse`，包含 `text`、`attachments`、`buttons` 等字段。
- 根据原消息的 `platform` 字段，将响应再次交给对应适配器的 `outgoing` 方法投递。

这个设计的优点是 agent 完全无平台概念，新增平台只需写一个适配器，不改动核心逻辑。

### 4. 会话隔离与去重

在统一消息中，`platform + chat_id` 构成了全局唯一的会话标识。agent 进行对话记忆或上下文管理时，使用该标识作为 key。这样一来，Telegram 群聊和 Discord 频道的对话互相独立，不会混淆。

去重方面，不同平台都可能出现重复投递（webhook 重试、事件重复触发）。利用 `platform + message_id` 去做幂等处理，比如在内存中缓存最近 1000 条已处理消息 id，或者在数据库中用唯一约束防止重复消费。

## 踩坑点

**消息长度与分片策略**
Telegram 文本上限 4096 字节，Discord 是 2000 字符。统一取较小值（2000）作为硬限制，超出则分段发送。但必须注意 Telegram 分片时仍要保持消息顺序，而 Discord 分段发送可能遇到速率限制，需要在适配器内加锁或延迟。

**富文本兼容性**
Telegram 支持 MarkdownV2 和 HTML，但某些特殊字符需要严格转义。Discord 默认使用 Markdown，但两者的转义规则不完全相同。如果 agent 需要输出格式化内容，最好让 agent 生成一种中间表示（如纯文本 + 格式标记），由各适配器负责渲染。否则会在跨平台出现显示错误或解析异常。

**速率限制与重试**
Discord 的全局速率限制非常严格，尤其是 bot 刚启动或大量发送时。需要在 `outgoing` 方法中实现全局 rate limit handler（`discord.py` 已内置，但自定义适配器需要自行处理）。Telegram 对 bot 发送消息的限制较宽松，但仍需在一定时间内避免狂奔，可以利用 `asyncio.sleep` 控制并发。

**文件与多媒体**
Telegram 对文件大小限制为 50 MB，Discord 为 8 MB（常规上传）或更大（Booster）。在适配器层先检查文件大小，超出则返回错误或压缩。

**安全和鉴权**
确保 Telegram webhook 端点仅 Telegram 可调用，Discord bot token 安全存储。对用户输入做清理，避免注入恶意格式化字符导致消息异常。

## 可复用建议

- **抽象适配器接口**：尽早将 “incoming/outgoing” 固化为接口，新平台实现该接口即可接入。
- **将平台配置外部化**：Telegram bot token、Discord bot token、webhook URL 等全部通过环境变量或配置文件注入。
- **保留原始消息引用**：在 `UnifiedMessage` 中存储原始 payload 的引用，便于平台特有处理（如 Discord 的交互式组件）需要访问原始事件。
- **使用消息队列解耦**：用 asyncio 队列或 Redis Stream 将接收与处理分离，可提高吞吐，避免 webhook 超时。
- **监控与日志**：记录每个平台的消息处理成功率、耗时、速率限制触发次数，便于定位瓶颈。

## 总结

跨平台消息路由的核心思路并不复杂：抽象统一消息协议，为每个平台编写适配器，然后让 agent 专注领域逻辑。用这样的方式，我把一个用于查询任务状态的自然语言 agent 同时部署到了公司内部的 Telegram 运维群和 Discord 团队频道，维护成本比预想的低得多。

新增平台时，只需要关注如何获得原始消息、如何把回复发回去，而无需修改任何 agent 代码。OpenClaw 社区的很多 agent 实现都可以通过这种模式快速实现多平台覆盖，避免陷入平台细节的泥潭。

---

