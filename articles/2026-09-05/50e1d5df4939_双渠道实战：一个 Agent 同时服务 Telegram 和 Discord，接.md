---
title: 双渠道实战：一个 Agent 同时服务 Telegram 和 Discord，接入不难，归一化才是工作量
feedId: 36211
source: 综合讨论
publishedAt: 2026-09-05
---

## 背景

我的 Agent 最初只挂在 Telegram 上，跑了小半年，skills 和 MCP 工具都堆在同一个实例里。后来团队协作搬到了 Discord，问题来了：再起一个 Agent 实例，意味着两份配置、两份会话、两份维护成本。目标很明确——一个 OpenClaw 网关，两个渠道，工具共享，会话不串。

## 问题

接第二个渠道本身不难，难的是三件事：**会话怎么隔离、消息格式怎么归一、两个平台的能力差异怎么兜底**。

## 做法

1. **同一网关挂双渠道**。配置里 telegram 和 discord 并列即可，不需要两个进程：

```json
{
  "channels": {
    "telegram": { "botToken": "<token>" },
    "discord":  { "token": "<token>" }
  }
}
```

2. **会话按渠道隔离**。会话键自带渠道前缀，天然不串。如果需要"同一个人跨平台共享上下文"，别默认合并，自己维护一张 user mapping 表做显式合并，可控得多。
3. **回复层做归一化**。Agent 内部统一输出受限 Markdown，发送前按渠道转换：Telegram 走 HTML parse mode（比 MarkdownV2 的转义规则省心太多），Discord 用原生 Markdown；长度分别按 4096 / 2000 切分，切分时优先保代码块完整。
4. **长输出不走消息体**。工具返回的大段结果落盘或生成摘要，只把结论塞进消息。
5. **网络路径**。Telegram 用 long polling，Discord 走 gateway websocket，两者都不需要公网回调，家庭宽带 + 一台内网机器就能跑。

## 踩坑点

- **Discord 的 MESSAGE CONTENT INTENT 没开**：bot 私聊正常、群里收不到正文，症状很迷惑。开发者后台一个开关的事，但要重启生效。
- **限流**：Discord 频道级限流比想象中严，任何 fan-out 广播逻辑必须过队列，否则全群静音。
- **群聊触发不一致**：Discord 靠 mention，Telegram 群靠命令或唤醒词，漏配一边就会出现"群里不理人"。
- **媒体时效**：两边图片 URL 有效期和获取路径不同，异步慢悠悠去下载会拿到失效链接，收到即落盘。
- **私信配对策略**：两边都要放行目标用户，否则 bot 表现为已读不回，翻日志才看到是被策略拦了。

## 可复用建议

- 渠道适配层保持"薄"，所有格式转换收敛到一个模块，以后加新渠道只写 adapter。
- 日志统一带渠道前缀，排障先看前缀再翻正文。
- 每个渠道做定时探活，挂了先报警再人工介入。
- system prompt 里不要写死渠道名，用变量注入当前渠道，让 Agent 知道"在跟谁说话"——措辞和长度策略可以因此不同。

## 总结

双渠道接入的连接成本，OpenClaw 基本替你付掉了；真正决定稳定性的，是会话隔离和格式归一这两层的纪律。建议顺序：先跑通纯文本收发，再补富文本和媒体，最后才加广播类功能——顺序别反。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/2ac696b2973712eb.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/fce3b84f98f60998.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/85d80f62c34a521e.png)

