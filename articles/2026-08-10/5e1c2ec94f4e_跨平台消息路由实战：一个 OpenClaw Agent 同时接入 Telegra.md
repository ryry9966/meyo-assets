---
title: 跨平台消息路由实战：一个 OpenClaw Agent 同时接入 Telegram 与 Discord
feedId: 32349
source: 综合讨论
publishedAt: 2026-08-10
---

## 背景

很多自动化实践者会遇到同一个问题：一个 Agent 写好以后，团队里有人用 Telegram，有人只用 Discord。如果维护两套 Bot，消息逻辑会迅速分裂，状态管理也会变成灾难。更理想的方式是让**同一个 Agent 实例**同时服务两个平台，用户无论在哪个平台发指令，都能得到一致响应，并且可以跨平台同步部分上下文。

OpenClaw 在设计上天然解耦了 Channel（接入渠道）与 Agent（执行核心），这给了我们做**多平台消息路由**的基础。但在真实落地时，依然有不少工程细节需要处理，尤其当你想保持长会话、处理平台差异、并统一异常回退时。

本文记录一次从零搭建的过程，目标是让一个 OpenClaw Agent 同时接入 Telegram 与 Discord，并做到：

- 同一套插件、工具、MCP 服务同时被两个平台复用
- 不同平台的输出格式自动适配（Markdown 降级、按钮映射等）
- 会话按“用户+平台”隔离，避免跨平台串扰
- 路由层足够薄，不侵入 Agent 核心逻辑

## 核心问题

将两个平台的 Bot 指向同一个 Agent，表面上只是再加一个 inbound adapter，实际上会遇到三类典型问题：

1. **消息格式不一致**  
   Telegram 支持 MarkdownV2 和有限的 inline keyboard，Discord 使用自己定制的 Embed 和交互组件。Agent 如果直接输出 Discord Rich Embed，Telegram 用户就会看到一堆 JSON。

2. **会话模型不同**  
   Telegram 的 chat_id 体系天然带群组维度，Discord 则分 Guild/Channel/DM 三层。如果直接把消息体透传给 Agent，会话管理逻辑会被污染。

3. **限频与重试差异**  
   Telegram 的 429 和 Discord 的 rate limit 行为不完全一样，如果在一个地方做统一重试，很容易误伤另一个平台。

因此需要一个清晰的路由适配层，把每个平台的差异性消化在 Agent 之外。

## 整体架构

```
┌───────────────┐
│  Telegram Bot │────Inbound Msg───┐
└───────────────┘                 │
                       ┌──────────▼──────────┐
                       │   Message Router    │
                       │  - 平台标识注入     │
                       │  - 格式标准化       │
                       │  - 会话 key 生成    │
                       └──────────┬──────────┘
                                  │ 标准化内部消息
                       ┌──────────▼──────────┐
                       │    OpenClaw Agent   │
                       │  - 插件 / 工具 / MCP │
                       └──────────┬──────────┘
                                  │ 统一响应
                       ┌──────────▼──────────┐
                       │   Response Adapter  │
                       │  - TG 格式化        │
                       │  - Discord 格式化   │
                       └──────────┬──────────┘
               ┌─────────────────┼─────────────────┐
        ┌──────▼──────┐   ┌──────▼──────┐
        │ Telegram Out│   │ Discord Out │
        └─────────────┘   └─────────────┘
```

关键设计决策：**Agent 只理解一种内部消息格式**，所有平台特性都在 Router 和 Adapter 中处理。这样做的好处是 Agent 零改动即可支持新平台，坏处是复杂交互组件（如模态框、表单）难以做到完全相同 —— 但对我们这类偏自动化、工具调用的场景影响不大。

## 实现步骤

### 1. 定义内部消息结构

在 OpenClaw 的上下文里，可以用简单的字典结构：

```python
{
  "platform": "telegram" | "discord",
  "session_id": "tg:12345:67890",  # 平台前缀 + 唯一用户/群标识
  "user_id": "user_abc",
  "text": "用户原始输入",
  "attachments": [{"type": "image", "url": "..."}],
  "metadata": {}  # 平台特有字段，仅透传用
}
```

`session_id` 是关键。我为 Telegram 使用 `tg:{chat_id}:{user_id}`，Discord 使用 `dc:{guild_id}:{channel_id}:{user_id}`。这样同一用户的 TG 私聊和 DC DM 互不干扰，但同平台同群内的上下文可以保持。

### 2. 构建 Message Router

Router 做三件事：

- 根据 webhook 来源打上平台标签
- 提取并标准化字段，生成统一的内部消息
- 注入给 Agent 执行

如果用 asyncio 驱动，Router 就是两个 webhook handler 的公共前置逻辑。以 Python 伪代码示意：

```python
async def handle_telegram(update):
    msg = normalize_tg(update)
    session_id = f"tg:{msg.chat_id}:{msg.user_id}"
    internal = {
        "platform": "telegram",
        "session_id": session_id,
        "user_id": str(msg.user_id),
        "text": msg.text,
        "attachments": extract_tg_attachments(msg),
        "metadata": {"message_id": msg.message_id}
    }
    response = await agent.process(internal)
    await send_tg(msg.chat_id, response)

async def handle_discord(message):
    msg = normalize_dc(message)
    session_id = f"dc:{message.guild.id}:{message.channel.id}:{message.author.id}"
    internal = {
        "platform": "discord",
        "session_id": session_id,
        "user_id": str(message.author.id),
        "text": msg.clean_content,
        "attachments": extract_dc_attachments(msg),
        "metadata": {"message_id": message.id}
    }
    response = await agent.process(internal)
    await send_dc(message.channel, response)
```

Agent 只调用 `process(internal)`，返回值也是统一的字典，包含 `text`、`embeds`、`components` 等字段，再由各平台的 Adapter 渲染。

### 3. 平台输出适配

Telegram 侧：

- `text` 使用 MarkdownV2 转义，注意 `. # ! ( ) -` 等特殊字符
- 按钮直接用 InlineKeyboardMarkup 构建
- 如果收到 `embeds`，降级为文本摘要 + 图片链接（因为 TG 不原生支持 embed）

Discord 侧：

- 直接用 `discord.Embed` 构建富文本
- 按钮转为 Discord 的 View + Button
- 注意消息长度限制（2000 字符），超出需要自动分片

统一响应的示例结构：

```python
{
  "text": "查询结果如下：",
  "embeds": [
    {"title": "服务器状态", "fields": [{"name":"CPU","value":"12%"}]}
  ],
  "components": [
    {"type": "button", "label": "刷新", "action": "refresh"}
  ]
}
```

响应适配器就是把这个结构翻译成具体平台的 API 调用。这样以后增加 Slack 或钉钉，只需要再加一个 Adapter。

## 踩坑记录

**坑1：Telegram MarkdownV2 转义地狱**  
Agent 返回的文本来自工具或 LLM，很可能包含未转义的特殊字符。如果你在发给 Telegram 前做一次全局转义，会把原本已经格式化的部分（比如手动加粗）二次破坏。建议的解法是：Agent 返回已经为 Telegram 转义好的 `tg_text` 字段，或者用 `entities` 方式直接构造格式化消息，绕开全局转义的局限性。这里我选择了后者，手动构建 `MessageEntity`，虽然代码量大一些，但可控性更好。

**坑2：Discord 的 interaction 超时**  
如果你在 Discord 里用按钮触发 Agent 再处理，Discord 要求 3 秒内给一个初始响应，否则会报 “This interaction failed”。异步 Agent 调用很可能超过这个时间。正确做法是先 `defer()`，等 Agent 跑完再 `edit_original_response`。但如果你同时有 Telegram 和内网 API 触发的调用，这个 defer 逻辑很容易遗忘。建议统一在 Agent 入口处检查交互类型，全局处理 defer。

**坑3：session key 冲突**  
一开始我直接用 `user_id` 做会话标识，结果发现不同平台的同一个用户（比如都用 Discord 和 Telegram，ID 可能不相关，但如果都用的同一个数字 ID 系统就会冲突）。加平台前缀后解决。如果你还要支持多租户，建议再加 tenant 前缀。

## 可复用建议

- **Agent 不要处理平台格式化**。哪怕目前只接入一个平台，也预留 adapter 接口，未来接入第二个只需加几十行代码。
- **统一限频处理**。不建议在 Agent 内部捕捉 429，而是让发送层各自处理。可以抽一个 `send_with_retry(platform, payload)` 的公共 helper，内部再根据平台选择不同的 backoff 策略。
- **平台特性映射表**。把“支持embed/不支持”“按钮数量上限”“消息长度上限”做成配置文件，避免代码里到处写 if platform == "telegram"。
- **测试路径要覆盖双平台**。至少准备一个 telegram mock 和一个 discord mock，确保回归时不偏废某一端。

## 总结

让一个 Agent 服务多个平台，本质上是把“渠道”从“业务”里剥离。OpenClaw 的 channel 抽象让这件事变得自然，但真正落地时，格式适配、会话隔离和限频处理需要仔细设计。我的经验是：轻量路由 + 内部标准消息 + 平台适配器，三者组合可以在不大幅增加复杂度的情况下，让 Agent 稳定地跑在 Telegram 和 Discord 上。

这套模式已经稳定运行了几周，支持文件分析、定时任务查询、状态看板等常用功能，两个平台的用户体验基本一致。如果你也在做类似的多平台实践，欢迎在社区分享你的路由设计。

---

