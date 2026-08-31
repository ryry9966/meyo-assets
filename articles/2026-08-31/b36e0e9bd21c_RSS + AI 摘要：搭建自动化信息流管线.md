---
title: RSS + AI 摘要：搭建自动化信息流管线
feedId: 35564
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景

RSS 依旧是最可控的信息输入方式之一，但订阅源一多，阅读压力主要来自长文、重复报道和低相关更新。与其每条都点开，不如让机器先读一遍，输出可决策的摘要。

## 问题

实操时，难点往往不在“调用 AI”，而在前处理和后处理：正文提取质量、去重、调度、失败重试、token 成本。目标不是生成学术级摘要，而是把一条信息压缩到 30 秒内判断“读不读”。

## 做法/步骤

### 1. 抓取层

使用 Miniflux 或 FreshRSS 作为 RSS 后端。它们提供 API、webhook 和过滤规则，避免自己处理解析与缓存。Miniflux 的 API 比较干净，适合脚本调用。

### 2. 正文提取

很多 feed 只给 description。用 trafilatura 抓原文，保留标题、作者、发布时间、原文 URL。失败时回退到 feed 自带的 summary。

### 3. 去重回退

按 canonical URL 或标题+发布日期做 hash，存 SQLite。处理状态标记为 pending / processed / failed。

### 4. AI 摘要

先截断正文到 2000 字符左右，避免 token 浪费。使用结构化输出，示例 prompt：

```json
{
  "summary": "一句话摘要",
  "points": ["要点1", "要点2", "要点3"],
  "relevance": "high|low",
  "need_read": true
}
```

要求模型只基于输入，不要编造。温度调低。

### 5. 输出层

high relevance 写入 Markdown 日报或推送 Telegram/邮件；low 只记录不推送。保留原文链接。

### 6. 调度

cron 每 30 分钟跑一次。脚本内部设置超时和重试。如果结合 OpenClaw，可把“摘要单条 URL”封装为 MCP tool，由 Agent 定时调用。

## 踩坑点

- **正文提取失败**：trafilatura 对 JS 渲染页面无能为力。需要 fallback 到 RSS description，并记录失败率。
- **重复推送**：同一文章带不同 utm 参数。去重应基于 canonical URL，而不是原始 URL。
- **AI 幻觉**：技术文里的版本号、命令可能被改。摘要必须附原文链接，不把 AI 输出当事实。
- **Token 成本**：全文投入会快速耗尽预算。先做规则过滤和截断。
- **编码问题**：中文 feed 偶发 GBK 乱码，统一按 UTF-8 处理，非法字符替换。
- **源不稳定**：feed 改版、证书过期会导致抓取失败。需要监控失败率，而不是静默跳过。

## 可复用建议

- 将 AI 摘要封装为 MCP server 的 tool，输入 URL 或文本，输出 JSON。这样 OpenClaw 的 Agent 可以复用，不绑定单一脚本。
- 分级处理：先关键词/正则初筛，再进 AI，减少调用量。
- 保留原始 JSONL：每天一份，方便重新摘要或调试。
- 记录处理条数、失败数、延迟和 token 消耗。
- 不要全自动：让 AI 输出 need_read 和建议，人最终决定。

## 总结

这条管线的核心不是“AI 替你读完所有东西”，而是把信息噪音降到可决策范围。稳定、可回放、可观测，比堆复杂功能更重要。对于 OpenClaw 用户，把它做成 MCP tool 后，信息流自动化的扩展空间会大很多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/0168cd2d92996c47.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/a9308bd6c22f8724.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/a533963a16904b94.png)

