---
title: 一个 Agent 核心，两个入口：Telegram + Discord 跨平台消息路由实践
feedId: 35891
source: 综合讨论
publishedAt: 2026-09-03
---

# 背景

我们在 OpenClaw 里跑着一个长期服务的 Agent，最初只挂在 Telegram 上。后来一部分用户活跃在 Discord，最省事的方案是复制一份配置再起一个实例——但很快发现问题：同一套 MCP 工具要维护两份、会话记忆完全分裂、用量统计对不上。于是把它重构成「一个核心、多个渠道适配器」的结构，这篇帖记录过程。

# 问题

拆开看其实是四件事：

1. **协议差异**：Telegram 走 Bot API（long polling / webhook），Discord 走 Gateway WebSocket，事件模型完全不同。
2. **消息模型差异**：回复、引用、附件、长度限制（Telegram 4096 字符，Discord 2000 字符）、Markdown 方言各不相同。
3. **会话归属**：不同平台的人怎么算同一个会话，需要明确决策。
4. **路由与限速**：两个渠道各有速率限制，回复乱序比回复慢更伤体验。

# 做法

核心决策只有一条：**Agent 核心不感知平台，所有平台逻辑压进适配器层。**

1. **定义统一信封（envelope）**。入站消息一律归一化为 JSON：`platform`、`chat_id`、`user_id`、`text`、`attachments`、`reply_to`、`capabilities`。`capabilities` 声明该渠道支持什么（embed、reaction、文件直传），核心按能力降级输出。
2. **两个适配器只做翻译**。Telegram 适配器负责 `update_id` 幂等去重；Discord 适配器负责 Gateway 重连和 MESSAGE CONTENT intent。适配器对内只产出/消费信封。MCP 工具层保持平台无关，reaction 这类平台特有能力通过 `capabilities` 门控注入。
3. **路由器按 `session_key = platform:chat_id` 串行化**。每个 session 一条 FIFO 队列，保证同一会话回复有序；跨 session 并行处理。
4. **出站走独立渲染层**。核心产出结构化内容（段落 + 可选附件），渲染层按平台拼格式：Telegram 用 HTML parse mode，Discord 用 Markdown，超长时按块边界拆条发送。
5. **跨平台身份默认不合并**。绑定做成显式命令，避免误把两个人的消息灌进同一上下文。

# 踩坑点

- **Telegram MarkdownV2 转义是灾难**：14 个字符要转义，漏一个就 400。换 HTML parse mode 后世界清净了。
- **Discord 拆分长回复**：2000 字符限制下不能把代码块拦腰截断，必须按块边界拆。
- **Telegram webhook 会重推**：失败重试不挡，就会出现重复回复，去重逻辑要在入口做。
- **Discord thread 串台**：thread ID 和频道 ID 语义不同，`session_key` 必须带上 thread 维度。
- **附件不通用**：Telegram 的 `file_id` 只在 Telegram 有效，跨渠道转发文件要落地重传，别偷懒传引用。

# 可复用建议

- 信封里加 `schema_version`，适配器升级不会炸核心。
- 用录制回放做测试：把真实信封存成 fixture，核心逻辑的回归测试不需要连外网。
- 观测按 hop 打点：入站 → 路由 → LLM → 渲染 → 出站，各段延迟分开看，定位快很多。
- 评估标准：新增平台 = 新增一个适配器 + 一个渲染器，核心零改动。这是这套结构真正的收益。

# 总结

跨平台路由的本质不是「多接几个 API」，而是消息模型归一化、平台差异关进适配器、会话状态收拢到核心。做完之后接第三个渠道（比如 Matrix）预计只要一个下午——这个判断等真验证了再回来更新。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/f00b612e3fb54ac1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/aec650dd7ef93169.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/9a7f906046e65029.png)

