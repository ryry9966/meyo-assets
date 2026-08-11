---
title: 跨平台消息路由实战：让一个 Agent 同时接住 Telegram 与 Discord 的消息
feedId: 32611
source: 综合讨论
publishedAt: 2026-08-11
---

## 背景

在自动化实践里，很多人最先把 Agent 部署到 Telegram，后来团队或社区又要求支持 Discord。两套 bot 代码各自维护，消息处理逻辑、限流策略、上下文管理都得写两遍，时间一长就成为负担。如果你的 Agent 核心业务逻辑已经稳定（比如基于 OpenClaw 插件体系构建的多轮对话、知识库检索、工具调用），完全可以让它通过一个统一的消息路由，同时服务多个聊天平台。下面就是一次工程化的落地过程。

## 问题

不同聊天平台的 API、消息格式、富文本语法、文件上传限制、速率限制差异很大。直接在一个进程里硬编码平台分支，会导致核心逻辑被平台细节污染。目标是：

1. 维护 **一个 Agent 核心**（OpenClaw 的 Processor/Plugin），只处理抽象消息；
2. 每个平台对应一个轻量适配器，负责平台消息 → 统一消息 → 平台回复；
3. 适配器与核心解耦，方便单独重启、扩容，也方便新增第三个、第四个平台。

## 做法与步骤

### 1. 设计统一消息模型

在 OpenClaw 体系中，通常已有 `Message` 和 `Response` 的抽象。需要扩展出几个共用字段：

```python
@dataclass
class UnifiedMessage:
    platform: str          # "telegram" | "discord"
    chat_id: str           # 唯一标识会话
    user_id: str           # 发送者 ID
    text: str
    attachments: List[Attachment]
    metadata: dict         # 透传平台特有字段
```

回复也类似，但需要携带原始 `chat_id` 以及调用的平台适配器标识，方便适配器将响应推回正确的对话。

### 2. 实现 Telegram 适配器

用 `python-telegram-bot` 快速启动一个轮询或 Webhook 的 bot。收到消息后，转换为 `UnifiedMessage`，通过消息队列（这里选用 Redis Pub/Sub）发布到 `agent:input` 频道。同时启动一个独立线程订阅 `agent:output:{platform}` 频道，当收到 Agent 核心处理后的回复，就调用 `send_message` 发回给用户。

关键转换：Telegram 的 MarkdownV2 与 Discord 的 Markdown 不完全相同，需要在适配器层面做一次格式清洗，确保纯文本能安全发给 Agent 处理。附件通过生成临时 URL 或本地路径传递给核心，避免大文件直接进队列。

### 3. 实现 Discord 适配器

用 `discord.py`，启用 `message_content` intent（需要 Bot 配置中开启）。做法类似：监听 `on_message`，组装 `UnifiedMessage`，publish 到同一个 `agent:input` 频道，同时订阅 `agent:output:discord`。要注意 Discord 消息可能超 2000 字符，适配器需要自动拆分长回复，或使用钩子函数生成 Embed/文件，这都属于平台特性封装。

### 4. Agent 核心订阅处理

核心是一个 OpenClaw 的 `Plugin`，启动后订阅 PUB/SUB 的 `agent:input`。收到统一消息后，走原有处理管线（意图识别、记忆管理、工具调用等），最终输出标准 `Response`。核心根据 `Response` 携带的 `platform` 字段，将回复 publish 到对应的 `agent:output:{platform}` 频道。这样无论增加多少个平台，核心代码一行不用改。

```python
# 核心伪代码
def handle(msg: UnifiedMessage):
    resp = self.processor.run(msg)   # OpenClaw pipeline
    channel = f"agent:output:{msg.platform}"
    redis.publish(channel, resp.to_json())
```

### 5. 运行结构

最后用 `supervisord` 或 `docker-compose` 分别拉起 `agent-core`、`adapter-telegram`、`adapter-discord` 三个服务。Redis 作为消息总线。适配器可以任意缩放，甚至部署在不同机器上，只要指向同一个 Redis。

## 踩坑点

- **消息 ID 与去重**：Telegram 的 `update_id` 和 Discord 的 `message.id` 范围不同，不能共用同一套去重缓存。适配器需要在消息入队前，用 `platform:message_id` 前缀去重。
- **格式解析**：Telegram 可能给用户消息中的 `/start` 等命令。Discord 可能识别不到这种纯命令。统一层可以把所有文本视为 `text`，交给 Agent 的 NLU 模型解析，而不是在适配器里硬编码指令。
- **速率限制**：Telegram 的 `retry_after` 和 Discord 的全局速率限制完全不同。如果适配器从队列中收到大批回复，必须自己实现漏桶或令牌桶，不能依赖 Agent 核心限流，否则会触发平台 ban。
- **连接恢复**：Redis 断连、Discord 网关重连，都要在适配器里处理。我们设置了指数退避重连，并确保重连后不会丢失 `agent:output` 的消息（切换为 Redis Streams 消费者组模式可以做到精确一次）。
- **文件上传**：两个平台的附件类型不完全重叠。适配器获取文件 URL 后，若目标平台不支持原样文件，适配器需要做转码或替换为链接，这步不能推到 Agent 核心，否则破坏纯粹性。

## 可复用建议

- 适配器要完全无状态（会话信息由 Agent 核心或外部存储管理）。
- 通过环境变量选择平台：`ENABLED_PLATFORMS=telegram,discord`，方便部署时裁剪。
- 所有平台特有逻辑（长消息拆分、Markdown 处理、附件转链）都封装成适配器内的 `Formatter` 插件，方便单元测试。
- 用 Redis Streams 替代 Pub/Sub，可以获得确认和重试能力，提高生产可靠性。
- 为每个平台写一个健康检查 HTTP 端点，便于监控和自动恢复。

## 总结

通过一套简单的消息路由模式，一个 OpenClaw Agent 的核心代码可以零改动地扩张到多个聊天平台。适配器只负责平台协议的脏活累活，Agent 核心保持干净。实际运行三个月，新增平台只需写一个百来行的适配器，核心管线稳定不变。如果你已经有 OpenClaw 的实践，不妨也把 Discord、Telegram 甚至是 Slack、Matrix 都接到同一个 Agent 上，维护成本会大大降低。

---

