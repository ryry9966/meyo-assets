---
title: 一个 Agent，两条战线：用 OpenClaw 搭建 Telegram+Discord 跨平台消息路由
feedId: 31795
source: 综合讨论
publishedAt: 2026-08-06
---

# 一个 Agent，两条战线：用 OpenClaw 搭建 Telegram+Discord 跨平台消息路由

## 背景：两个 Bot 不如一颗大脑

社区团队经常会同时维护 Telegram 和 Discord 作为用户通道——一个偏即时问答，另一个承载长讨论。如果给每个平台写一个独立的 Bot，不仅业务逻辑要维护两份，回答的一致性也难保证。更头疼的是，后续接搜索、数据库、外部 API 这些能力时，任何一个改动都要在两个项目里同步。

于是很自然想到：能不能用一个 Agent 的大脑去驱动两个平台的“身体”？只要它能理解“这是来自 Discord 频道的一条消息”，处理完后再把结果的格式和长度适配回来源平台，这套东西就能跑。

这篇文章会记录我在 OpenClaw 上做这件事的实际做法——从消息抽象、路由调度，到速率限制和会话隔离的踩坑细节。

## 问题拆解：要解决的不只是格式

跨平台消息路由表面上只是“接两个 webhook，再发回结果”，但真正落下去会发现三个躲不开的问题：

1. **消息模型不统一**：Telegram 支持 MarkdownV2 和 Telegram entities，Discord 有自己的消息体结构，富文本和附件处理差异显著。
2. **平台侧限制不同**：Discord 的全局速率限制非常严格（每 10 秒约 50 条消息），Telegram 则宽松得多；消息最大长度、文件大小限制也完全不同。
3. **会话与上下文隔离**：不能因为同一个微信号在 Telegram 和 Discord 都发了消息就把上下文混在一起；需要以「用户+平台」作为最小隔离粒度。

## 做法：三层适配，一处大脑

整个实现可以抽象成三层：**消息适配层 → 路由调度层 → Agent 处理层**。每一层只做自己职责内的事。

### 1. 统一消息模型
先定义一套平台无关的内部消息体，例如：

```python
@dataclass
class UnifiedMessage:
    platform: str          # "telegram" | "discord"
    channel_id: str        # 群组/频道 ID
    user_id: str
    user_name: str
    text: str              # 纯文本，不含平台格式
    attachments: list[str]
    raw: Any               # 原始消息对象，必要时引用
```

两端的适配器收到 webhook 后，只负责把平台原始消息填进这个结构体。这个模型足够薄，不喜欢 dataclass 的也可以直接用 TypedDict。

### 2. 平台适配器
Telegram 端我用 `python-telegram-bot` 的 `Application` 接 webhook，Discord 端用 `discord.py` 的 `commands.Bot` 同样配置为 webhook 模式（避免长时间持有 websocket 连接的问题，便于部署）。两个适配器都只做三件事：

- 解析 webhook 请求并进行签名校验（Discord 需要验证公钥，Telegram 校验 token）
- 将消息转为 `UnifiedMessage`
- 调用统一的 `route_message(msg)`，并把返回的文本**逆向适配**后发回对应平台

### 3. 路由调度与 Agent 入口
`route_message` 这一层负责：

- 生成 `session_key = f"{msg.platform}:{msg.user_id}"`，用于上下文隔离
- 构造 Agent 的输入（可以带历史上下文）
- 调用 OpenClaw 内部的同一个 Agent 实例处理
- 拿到 Agent 响应后，**长消息智能拆分**，并在必要时做格式降级（比如 Discord 不支持的部分 Telegram Markdown 语法转成纯文本或简单 emoji）

有时还需要加上延迟发送和队列控制来应对平台速率限制。我是用 `asyncio.Queue` 实现了一个单平台限速队列，Discord 端限制每 1 秒最多 5 次 API 调用，Telegram 保持默认 30 次/秒。

### 4. MCP 工具集成（可插拔扩展）
Agent 需要外部能力时，我用 OpenClaw 的 MCP 客户端接入了搜索引擎和内部 wiki。因为消息适配层已经把平台差异屏蔽掉，Agent 只看到标准输入，工具调用完全不用关心最终回的是哪个平台。以后加 Slack、飞书，也只是增加一个适配器。

## 踩坑实录

- **富文本格式转换是坑中之坑**：Telegram 的 `bold` 用 `*text*`，Discord 用 `**text**`；Telegram 支持部分 HTML 实体，Discord 不支持；最稳妥的办法是「纯文本优先」，只在确认目标平台支持时做富文本转换，否则退回纯文本。这就是为什么 `UnifiedMessage.text` 我强约束为纯文本，响应文本也先构造纯文本，再由适配器做可选的上标转换。
- **Discord 速率限制比文档写的还狠**：文档说 50 条/10 秒，实测在群聊密集时更早遇限。我的做法是加一个全局 `Semaphore`，限制并发 API 请求数，并在收到 429 时自动 retry。retry 时注意读取 `Retry-After` 头。
- **上下文长度不能平摊**：两个平台的会话如果都保留长历史，token 消耗疯涨。最终我只保留最近 6 轮上下文，且不同平台共享同一个 Agent 但各自独立的 `ConversationMemory`。
- **长消息拆分的“断句”问题**：简单按字符数拆分容易在链接或代码块中间断掉。我的做法是优先在换行符处拆分，若没有则按空格，实在找不到才按字符截断，并加上 `(continued)` 提示。

## 可复用建议

1. **将平台适配定义为 `Provider` 接口**，包含 `parse_incoming` / `send_outgoing` 等方法。新平台接入只需实现这个 interface，不用碰路由和 Agent 代码。
2. **统一消息模型尽量做薄**，富文本和文件引用都延迟到适配器处理，不要试图在中间层承载完整渲染逻辑。
3. **速率限制、重试、长消息拆分这些通用逻辑下沉到路由层**，用中间件方式组合，降低适配器心智负担。
4. **Agent 上下文隔离用 `platform:user_id` 作为 session key**，再加一个 `user_identity` 表做跨平台关联（可选），但只在用户主动绑定时启用，不自动合并，避免隐私风险。
5. **工具调用时注意输出长度限制**，Agent 生成的搜索摘要可能过长，路由层需要做好截断和提示。

## 总结

跨平台消息路由的本质，是在统一的大脑和不同的手脚之间做一次负责任的翻译和调度。只要把消息抽象和适配器做扎实，平台差异就不会渗透进 Agent 的业务逻辑。OpenClaw 的插件机制让这一步变得很轻——MCP 工具可以被任意平台复用，新平台的适配器也可以快速插拔。

对于已经有稳定用户群的小团队，这套方案省下的不止是代码量，更是维护两个 Bot 时偷偷积累的大量上下文漂移和响应不一致。

---

