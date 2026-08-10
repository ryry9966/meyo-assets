---
title: 同一个 Agent 同时服务 Telegram 与 Discord：消息路由与适配实践
feedId: 32498
source: 综合讨论
publishedAt: 2026-08-11
---

## 背景

在 Agent 应用落地的过程中，一个常见需求是让同一个智能体同时存在于多个即时通讯平台，比如既有 Telegram 用户，又有 Discord 社区。如果为每个平台各写一套 bot 逻辑，不仅维护成本高，状态与知识也无法共享。我们希望 **一个 Agent 实例** 能够统一处理来自 Telegram 和 Discord 的消息，返回风格一致的回复，同时又能正确适配不同平台的格式约束。

OpenClaw 生态中，我们可以借助 **MCP (Model Context Protocol)** 的 transport 抽象和插件化的平台适配器，相对干净地解决这个问题。本文分享一种生产可用的跨平台消息路由方案，包含架构、具体步骤、踩坑点和复用建议。

## 问题拆解

要实现一个 Agent 同时服务两个平台，有三个核心问题需要解决：

1. **消息格式异构**  
   Telegram 使用 HTML/MarkdownV2 有限的标记集，Discord 使用 Markdown 但有自己的一套转义规则，两者对图片、按钮、嵌入式消息支持完全不同。
2. **会话与用户身份映射**  
   同一个人类用户可能在两个平台都有账号，Agent 需要区分但不混淆；同时，不同平台的会话模型（Telegram 的 chat_id，Discord 的 channel/guild）需要统一抽象。
3. **消息处理与限速**  
   Telegram 和 Discord 的 API 速率限制、消息长度限制各不相同，Agent 的回复需要根据平台裁剪，且不能因一个平台的限速阻塞另一个。

## 方案设计

整体架构如下：Agent 内核只接收和产出 **标准化的内部消息（NormalizedMessage）**，与具体平台无关。每个平台通过一个 **Platform Adapter** 负责两件事：

- **Ingress**：将平台原生消息转换为 NormalizedMessage，并注入平台标识与用户 ID。
- **Egress**：将 Agent 产出的 NormalizedMessage 转换并发送到对应平台。

```
[Telegram Bot] ──> Adapter (ingress) ──> Message Queue ──> Agent Core
[Discord Bot]  ──> Adapter (ingress) ──> Message Queue ──> Agent Core
Agent Core ──> NormalizedMessage ──> Adapter (egress) ──> Platform
```

Agent Core 通过 MCP 在同一个进程中暴露工具和推理能力。每个 platform adapter 是一个独立模块，可以独立启动和配置，但共享同一个消息队列和 Agent 实例。这样即使某个平台连接断开，也不会影响其他平台的消息缓存和后续重试。

## 实现步骤

### 1. 定义内部消息模型

使用一个简单的 NormalizedMessage 结构，关键字段：

- `platform`: `"telegram"` 或 `"discord"`
- `channel_id`: Telegram 的 chat_id 或 Discord channel id
- `user_id`: 平台用户唯一 ID（Telegram 的 user id，Discord 的 author id）
- `text`: 纯文本内容（已去除平台标记）
- `attachments`: 统一后的附件列表（URL + 类型）
- `reply_to`: 可选的引用消息 ID

### 2. 编写 Platform Adapter

对于 Telegram，我们使用 `python-telegram-bot` 库。Ingress 部分在消息 handler 中将 `update.message` 转换为 NormalizedMessage，放入一个 `asyncio.Queue`。Egress 部分监听另一个队列，根据 NormalizedMessage 的 `text` 和可能的附件，调用 `send_message`，并处理 MarkdownV2 的转义（如 `_ * [ ] ( ) ~ > # + - = | { } . !` 必须转义）。

对于 Discord，使用 `discord.py`。Ingress 在 `on_message` 事件中构造 NormalizedMessage，同样放入队列。Egress 需处理 2000 字符限制，以及 Discord 的 `embed` 对象。简单做法是超过 2000 字符自动分段发送，或压缩为文件上传。

### 3. Agent 统一消费

Agent Core 从 ingress 队列中获取消息，根据 `user_id` 管理上下文（可在 memory 模块中按 user_id 隔离）。处理完毕后，将产生的 NormalizedMessage 放入 egress 队列。Platform Adapter 只从队列中取自己平台的消息（通过 `platform` 字段过滤），并执行平台特定发送逻辑。

### 4. 身份与权限控制

可以维护一张 `identity_mapping` 表，将 Discord 用户 ID 与 Telegram 用户 ID 关联（通过一个已有的认证流程）。Agent 在做个性化回复或权限判断时，依赖统一的内部 user_id，使得跨平台身份可识别。

### 5. MCP 工具复用

所有工具（如查询数据库、调用 API）均通过 MCP 工具暴露，Agent Core 调用时不感知平台差异。工具返回结果可能包含格式化内容，Adapter 负责根据平台能力简化（如 Discord 不支持某些交互式组件，就转为纯文本提示）。

## 踩坑记录

**1. 消息转义地狱**  
Telegram 的 MarkdownV2 要求几乎所有标点转义，且不允许嵌套粗体与斜体。Discord 的转义规则不同（反引号、星号、下划线等）。我们最终选择让 Agent 只输出纯文本或一种受限的抽象标记（如 `**bold**`），然后在 Adapter 层为每个平台做转换。但这种方式在表格、列表等复杂内容上依然受限。一个折中是 Agent 输出用极简 Markdown，两地 Adapter 各自做“安全清洗”。

**2. 长文本截断策略**  
Discord 消息限制 2000 字符，Telegram 是 4096。粗暴截断可能破坏 Markdown 配对，导致渲染错乱。所以我们实现了一个小状态机，在边界附近找到最近的段落结束处，安全分段。Telegram 侧则可以利用 `parse_mode` 开关，回退到纯文本以保证送达。

**3. 速率限制与阻塞**  
如果 Agent 短时间内大量回复，一个平台的 429 错误可能导致整个 egress 队列阻塞。解决办法是让每个 Adapter 的发送逻辑自带指数退避和单独的任务队列（使用 `asyncio.Semaphore`），不互相等待。

**4. 会话状态污染**  
同一个用户在两平台同时发消息，Agent 需要正确处理并发。我们为每个 `user_id` 加锁或使用对话级队列，避免 A 平台的问题插入到 B 平台的对话上下文中。

## 可复用建议

- **Platform Adapter 抽象类**：定义一个 `BaseAdapter`，包含 `ingress()`、`egress()`、`format_message()` 方法。新增平台时只需实现该适配器，Agent 内核零改动。
- **消息 ID 追踪**：在 NormalizedMessage 中保留 `platform_message_id`，可用于后续编辑、删除、反应等高级操作，方便适配器处理 Discord 的 webhook 交互。
- **配置化平台通道**：使用 OpenClaw 的配置系统，按 `platform: channel_id` 维度配置哪些群组/频道启用 Agent，方便灰度或关闭。
- **监控与告警**：为每个 Adapter 的 ingress/egress 队列积压、发送失败率设置指标，便于发现某平台异常。

## 总结

通过引入平台适配层和统一消息模型，可以使一个 Agent 核心同时可靠地服务 Telegram 和 Discord，且保持代码结构清晰、易于扩展。真正工程化的难点不在 Agent 本身，而在消息格式的差异适配和错误处理。对于 OpenClaw 用户，这套方案可以直接利用 MCP 的 transport 和插件体系落地，几分钟就可以让同一个智能体出现在多个社区里，且行为一致、记忆共享。

---

