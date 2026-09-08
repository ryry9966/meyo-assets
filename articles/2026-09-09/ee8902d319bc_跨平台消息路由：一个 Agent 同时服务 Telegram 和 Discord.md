---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 36665
source: 综合讨论
publishedAt: 2026-09-09
---

## 背景

我们的 Agent 最初只挂在 Telegram 上，服务一个内部群。后来发现另一拨协作者整天泡 Discord，同样的通知和问答要在两边人肉搬运。目标很明确：跑一个 OpenClaw 实例、维护一份 prompt 和记忆，同时接住两个平台的消息。

## 问题

这不是"多开一个 channel"那么简单，实际有三个矛盾：

1. **会话边界**：完全共享 session 会串台（Telegram 里的上下文漏进 Discord 回复）；完全隔离又丢掉统一记忆。
2. **格式差异**：Telegram 单条上限 4096 字符，Discord 是 2000；两边 Markdown 方言不同，表格、引用渲染各玩各的。
3. **触发模型**：两边都是长连接收消息，网关要能同时稳定持有两条链路，且群聊触发条件不一致。

## 做法

1. **准备凭据**：BotFather 拿 Telegram token；Discord Developer Portal 建 Application，**务必打开 Message Content Intent**。
2. **启用 channel**：Telegram 内置；Discord 装插件并启用：

```bash
openclaw plugins install @openclaw/discord
```

3. **配置网关**：在 `~/.openclaw/openclaw.json` 的 `channels` 下分别写 `telegram` 和 `discord` 两段，token、allowlist、DM 策略、群聊 mention 要求分开配。
4. **会话策略**：我们选了折中方案，**session 按平台隔离，workspace 记忆共享**。两边对话上下文互不污染，但 `AGENTS.md` / `MEMORY.md` 是同一份，人格和长期记忆一致。
5. **输出规范化**：系统提示里明确"不用表格、避免嵌套引用"，长回复配置分段，把格式适配交给 channel 层。
6. **验证**：重启网关后 `openclaw channels status` 确认两个 channel 在线，各端实测收发。

## 踩坑点

- Discord 忘开 Message Content Intent：事件能收到但**正文永远是空的**，排查了半小时才反应过来。
- Telegram 之前被设过 webhook，polling 拉不到消息，需要先清掉 webhook 再起网关。
- 没做分段的长回复在 Discord 被硬截断，尾部代码块没闭合，渲染直接炸了。
- Telegram 群默认 privacy mode 只响应命令或 mention；Discord 要 @bot 或配 prefix。两边触发条件不同，表现为"时灵时不灵"。
- 双平台同时高频使用会撞 Discord rate limit，广播类任务要加队列和退避。

## 可复用建议

- **路由和格式适配压在 channel 层，别写进 prompt**。prompt 只描述"内容应该长什么样"。
- "记忆共享 + session 隔离"是实践中最省心的折中，除非你有明确的跨平台续聊需求。
- 每个 channel 独立 allowlist 和独立日志，权限和排障都干净。
- 一个网关进程跑两个 channel，好过两个实例各挂一个——工具连接（包括 MCP server）只需维护一份。
- 上线前用小号跑一轮标准用例：短文本、长文本、图片、语音、群聊 mention，两端各过一遍。

## 总结

跨平台的核心不是把配置复制两份，而是想清楚：**哪里必须一致**（人格、记忆、工具），**哪里必须隔离**（session、权限、格式）。OpenClaw 的 channel 抽象基本够用，坑大多在平台侧的细节约定上。把这套结构跑稳之后，接第三个平台基本是原样复用。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/65c4394e6660f373.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/2c5d174c91f5b7ee.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/723df41afeb01c40.png)

