---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 36869
source: 综合讨论
publishedAt: 2026-09-10
---

## 背景

我们社区的用户一半在 Telegram，一半在 Discord。最初的做法是部署两个 Bot 实例，各自接一个 Agent。问题很快出现：两边记忆不共享、配置要改两遍、排障时在两套日志之间来回切换。后来我们把架构收敛成"一个 Agent 内核 + 多个渠道适配器"，这篇帖子记录这次改造的思路和踩坑。

## 问题

跨平台路由的核心不是"把消息转发过去"，而是三个更细的问题：

1. **会话归属**：同一用户在两个平台发消息，算一个会话还是两个？
2. **消息形态差异**：Telegram 的 MarkdownV2 转义规则和 Discord 的 Markdown 子集完全不同；Discord 有 thread 和 embed，Telegram 有 reply_to 和 topic。
3. **回复通道**：回答必须原路返回，不能串台——尤其当多个渠道的流量在同一个进程里并发时。

## 做法

架构分三层：

**适配层**。Telegram 和 Discord 各写一个 adapter 插件，职责只有一个：把平台原始事件归一化成统一信封：

```json
{
  "platform": "telegram | discord",
  "chat_id": "...",
  "user_id": "...",
  "msg_id": "...",
  "content": "...",
  "media": [],
  "reply_to": null
}
```

**路由层**。信封进入路由器，按 `session_key = hash(platform, chat_id, thread_id?)` 生成会话键，投递到对应 Agent 会话。关键决策：**跨平台不共享会话**。同一个人在两边就当两个用户，理由是上下文风格差异大，强合并反而产生答非所问。如果确实要打通，用显式绑定命令（用户主动发起身份关联），而不是默认合并。

**输出层**。Agent 产出的回复是平台无关的中间格式（段落 + 代码块 + 长度标记），由 adapter 侧的 renderer 翻译成平台原生格式。超长内容在输出层截断并附"续读"附件，不让 Agent 自己操心平台限制。

工具层（MCP）完全复用，Agent 内核感知不到底下是哪个平台。

## 踩坑点

- **MarkdownV2 转义**：Telegram 对 `_*[]()~` 等字符的转义要求近乎苛刻。最终放弃让 LLM 直接输出 MarkdownV2，统一由 renderer 做转义，LLM 只输出宽松 Markdown。
- **Discord 长度限制**：2000 字符比 Telegram 的 4096 紧得多，长回答必须切片或转附件。切片时注意别把代码块劈成两半。
- **并发串台**：早期路由器用全局队列，两个平台消息交错时偶尔回错频道。改成按 session_key 分片后解决。教训：**回复必须携带来源信封的引用，路由器只信信封不信时序**。
- **限流差异**：Discord 的 rate limit 按桶计算，群发通知一上线就撞墙。适配层要各自实现独立的出站队列和退避。
- **媒体不对称**：Telegram 的贴纸、语音在 Discord 侧没有对应物。归一化时明确降级策略（丢弃并提示），别让 Agent 幻觉出对媒体内容的描述。

## 可复用建议

1. 先定义消息信封 schema，再写 adapter——schema 是现在的你和未来的你之间的契约。
2. 输入归一化，输出平台化渲染，中间的 Agent 保持平台无关。
3. 每条出站消息记录 `(platform, chat_id, msg_id)` 三元组，排障时能快速还原"哪条进、哪条出"。
4. 跨平台身份合并做成显式 opt-in，永远不要默认合并。

## 总结

改造后，新增一个渠道（比如 Slack）的成本降到"写一个 adapter + 一个 renderer"。核心经验一句话：**平台差异关在适配层里，Agent 只面对统一信封**。路由逻辑本身很简单，难的是克制——不要在 Agent 里写任何 `if platform ==` 的分支代码。遇到新平台需求时，先问自己：这个问题能不能在信封 schema 或 renderer 里解决？大多数时候，答案是能。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/4a1645f7e7ee55e0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/81a3d7efce3b5b47.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/4a4c8315faa1270f.png)

