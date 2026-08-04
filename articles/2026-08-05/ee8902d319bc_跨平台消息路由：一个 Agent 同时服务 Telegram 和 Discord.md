---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 31665
source: 综合讨论
publishedAt: 2026-08-05
---

## 背景

社区里跑通了 Agent 之后，下一个问题通常是：怎么让更多人用上？

拉到 Telegram 和 Discord 是两个常见方向。但很多人一开始的做法是跑两个 bot 实例，一份代码部署两遍，prompt 改一处忘另一处，工具配置漂移，最后维护成本比写功能还高。本文记录一种更省事的方式：用 OpenClaw 把 Agent 做成单一后端，Telegram 和 Discord 作为对等通道接入，由路由层负责消息进出的归一化。

## 问题

本质问题是三个：

1. **状态割裂**：两个 bot 各自维护会话状态，用户跨平台问同一件事，Agent 不记得上下文。
2. **配置漂移**：MCP 工具、系统提示词、模型参数，双份副本必然不同步。
3. **开发提速受阻**：修一个 bug 要改两处，加一个命令要发两次。

## 做法

核心思路：**Agent 只做一件事，渠道交给路由层**。

具体拆成三层：

**第一层：渠道接入（Adapter）**
Telegram 和 Discord 各自通过 Bot API / Bot Token 接入 OpenClaw。这里 OpenClaw 的 Gateway 天然支持多渠道注册，不需要引入外部消息中间件。

**第二层：消息归一化（Normalizer）**
把 Telegram 的 `Message` 和 Discord 的 `Message` 转成统一结构：

```
{
  platform: "telegram" | "discord",
  channel_id: string,
  user_id: string,
  text: string,
  attachments: [],
  reply_to: string | null
}
```

这样下游 Agent 不需要关心消息来自哪个平台。

**第三层：统一 Agent 处理**
OpenClaw 启动一个 Agent 实例，注册 MCP 工具、加载系统提示词。所有消息进入同一处理管线，结果再按目标渠道的格式输出（Telegram 支持 Markdown 语法与 Discord 略有差异，由输出适配器处理）。

配置上，大致思路：

```yaml
gateway:
  channels:
    - type: telegram
      token_env: TELEGRAM_BOT_TOKEN
    - type: discord
      token_env: DISCORD_BOT_TOKEN

agent:
  model: deepseek-chat
  system_prompt: "你是跨平台助手..."
  mcp_servers:
```

## 踩坑点

**1. 平台审核要求**
Discord 对机器人有认证要求，超过一定规模需要申请验证，Telegram 相对宽松。开发阶段无所谓，但进入正式环境前要提前走流程。

**2. 身份映射**
两边的 `user_id` 格式不一样，Telegram 是数字 ID，Discord 是 Snowflake。如果不做映射，同一个用户跨平台会被当成两个人。建议在 OpenClaw 的 key-value store 里存一张 `platform_user_id -> internal_user_id` 的表，内部统一用后者。

**3. 速率限制**
Discord 的 rate limit 比 Telegram 严格得多。如果 Agent 返回长文本时分段发送，很容易触达 Discord 的每通道每秒限速。稳妥做法是对 Discord 输出做队列串行化，Telegram 保持并发。

**4. 消息格式差异**
双向不同：Telegram 的 Markdown 支持有限，Discord 用 `__underline__` 和行内代码块时要有转义处理。建议所有富文本在归一化时降级为纯文本 + 简单 Markdown，富媒体按平台能力增强。

**5. 编辑事件**
Discord 用户编辑消息会触发 `message_update`，Telegram 没有等价物，但如果用 inline keyboard 回调，事件类型也不一致。路由层要忽略或统一处理，否则 Agent 会收到重复/冲突输入。

## 可复用建议

- **先定协议再写代码**：消息结构先定好，Telegram 和 Discord 适配器只是实现细节。
- **日志打上 platform 标签**：排查问题时能快速区分是"Agent 逻辑问题"还是"渠道适配问题"。
- **抽象输出模板**：需要返回列表、代码块、链接时，写统一的模板函数，不要在两套代码里各写一遍。
- **灰度发布**：先在 Discord 群组小范围跑，观察错误日志，稳定后再放 Telegram 大群。

## 总结

不是所有场景都需要双平台。如果你的目标用户集中在单一社区，跑一个渠道就够了。但如果你需要覆盖不同人群（开发者常驻 Discord，运营/商务常驻 Telegram），单体 Agent + 多通道路由是维护成本最低的方案。OpenClaw 的 Gateway 已经把大部分渠道差异挡在门外，剩下的是业务层的归一化和平台细节处理。

核心收益是：**一套 Agent 逻辑，多端触达，少一半维护工作量**。

---

