---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 31548
source: 综合讨论
publishedAt: 2026-08-04
---

## 背景

社区用户往往同时活跃在 Telegram 和 Discord 上：Telegram 适合快速同步、频道广播，Discord 则承载了更丰富的主题子频道和语音场景。早期我们为每个平台各部署了一个 Agent 实例，结果是上下文割裂：用户在 Telegram 里问过的问题，切到 Discord 再问一遍，Agent 完全不记得。维护两份配置、两套系统提示词，也显著增加了心智负担。

方案很自然：抽出一个统一的消息路由层，让一个 Agent 核心同时对接两个平台的 Bot API。本文记录的是实际落地过程中的架构选择与踩坑点，不涉及任何特定框架，核心思路可以复用到你自己的 OpenClaw/MCP 插件组合里。

## 问题拆解

要让一个 Agent 同时服务两个平台，真正要解决的是三个问题：

1. **消息同构化**：Telegram 和 Discord 的消息模型不同（chat_id 语义、消息类型、附件元数据各有差异），路由层必须把它们归一成统一的内部事件结构。
2. **会话状态隔离**：同一个 Agent 进程要能区分「这是 Telegram 群里 @ 的消息」和「这是 Discord 私聊的消息」，会话 Key 必须带上平台维度。
3. **出口分发与失败隔离**：Agent 消费统一事件后，回复要按原平台原路返回；任何一个平台的 API 抖动不能阻塞另一个平台的消息处理。

## 做法与步骤

### 1. 定义统一消息事件结构

内部约定一个最小化的消息 Schema，例如：

```python
@dataclass
class UnifiedMessage:
    platform: str          # "telegram" | "discord"
    chat_id: str           # 归一化后统一的会话标识
    user_id: str
    text: str
    attachments: list      # 统一后的附件描述
    raw: dict              # 保留原始负载，便于排查
```

两个适配器各自负责把平台消息翻译成这个结构。关键点是 `chat_id` 的归一化：Telegram 的 chat_id 是整数，Discord 的 channel_id 与 guild_id 需要拼接。我们统一把它们转成字符串，Discord 侧用 `"{guild_id}/{channel_id}"`，Telegram 侧直接用 `str(chat_id)`。

### 2. 适配器各自独立收发

用 `python-telegram-bot` 和 `discord.py`（或你熟悉的异步 SDK）各起一个后台任务：

- Telegram 采用长轮询（Polling），避免公网暴露 Webhook 端点
- Discord 使用 Gateway 长连接，自带重连机制

两个适配器完全独立，各自维护自己的事件循环，只通过一个进程内的 `asyncio.Queue` 向路由核心投递消息。

### 3. 路由核心维护会话上下文

路由核心维护一个 `dict[tuple[str, str], ConversationState]`（元组即 `(platform, chat_id)`）。收到消息后：

1. 按 (platform, chat_id) 取或建会话状态
2. 将 UnifiedMessage append 到上下文
3. 调用 Agent 核心（LLM 调用或你的 Agent/OpenClaw 流程）
4. 将回复通过原适配器的发送接口返回

### 4. 出口统一调度

每个适配器暴露一个 `send_message(chat_id, text, files)` 接口，内部各自实现平台特定的格式化。路由核心不关心发送细节，只负责把 Agent 输出派发回正确的适配器入口。

## 踩坑点

**格式化语法差异**：Telegram 支持 HTML 和 MarkdownV2，Discord 只支持自己的子集 Markdown。同一个 Agent 回复字符串直接透传，会在 Discord 上显示裸露的 `*` 或者失效的链接。解决方式是路由层对 Agent 输出做一次平台适配：维护一个轻量的「文本后处理」函数，按平台转换加粗、斜体、行内代码的写法。

**Discord 的 2000 字符限制**：Agent 生成长文时容易被 Discord 截断且无提示。我们在路由层增加了分片逻辑：超过 1900 字符时按段落切割为多条消息依次发送，Telegram 则无需分片（上限 4096 但实际够用）。

**速率限制不对称**：Telegram 对单 chat 的消息频率限制较宽松，Discord 对 bot 的每条消息有严格的 rate limit。如果两个平台的消息混在同一个队列里消费，Discord 的重试逻辑会把 Telegram 的消息也拖慢。必须让出口调度按平台分队列。

**附件处理差异**：Telegram 允许最大 50MB 的本地文件 URL，Discord 默认只有 8MB（虽然可以通过配置提高）。适配器收到附件时，需要先归一化保存到本地临时目录，再按目标平台的上传约束处理。

**Webhook 与轮询的选择**：早期 Telegram 用了 Webhook，内网穿透不稳定导致消息丢失。本地轮询反而更可靠——在没有公网 IP 的开发环境里，这几乎是必踩的坑。

**本地优先的测试策略**：实测中，把两个适配器与路由核心解耦后，可以在本地用一个独立的「模拟客户端」注入 UnifiedMessage，完全不需要真实 Bot Token 就能跑通 Agent 核心的测试。这大幅降低了调试成本。

## 可复用建议

- **先做协议层，再做功能层**：路由层只负责「消息通路」，Agent 业务逻辑完全不知道对方是 Telegram 还是 Discord。后续接入 Slack 或 Matrix 只需新增适配器。
- **所有日志带 platform 前缀**：排查问题时，`[telegram]` / `[discord]` 前缀能让你在混合日志里三秒定位。
- **每个平台独立异常处理**：Discord 断线重连期间，Telegram 的用户不应感知到任何延迟。
- **起步阶段把状态放在内存**：单机单进程跑通后再考虑 Redis 持久化上下文，不要一上来就上重中间件。

## 总结

跨平台消息路由的本质是「统一入、平台化出」。用一个归一化消息事件做输入抽象，用平台适配器做输出适配，Agent 核心只面对一种数据形态。这套做法让后续新增平台变成了纯粹的协议翻译工作，也让 Agent 的上下文真正跨平台延续。架构不复杂，但平台的边界条件（格式化、限速、体积限制）才是真正的工程成本所在。

---

