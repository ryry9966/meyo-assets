---
title: 用 OpenClaw 搭一条 RSS + AI 摘要管线：从抓取到投递的工程化实践
feedId: 35352
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景

RSS 适合持续追踪信息源，但每天几十上百条未读，真正值得读的很少。把 RSS 抓取、去重、AI 摘要、定时投递串成一条自动化管线，能有效降低信息过载。本文面向已经在 OpenClaw、Agent、MCP、插件体系里做自动化的用户，不堆概念，只讲可复现的落地方式。

## 问题

直接“抓完丢给大模型”通常会遇到几类问题：

1. 清洗不干净：RSS 的 `summary` / `content` 字段混有 HTML、脚本、广告和样式。
2. 重复消费：同一 item 多次抓取，或同一内容被多个源转发。
3. 摘要质量不稳定：模型容易复述标题，或者对长文简单截断后丢失重点。
4. 管线脆弱：RSSHub 源挂掉、feed 编码异常、限频、时区错位，任何一环都可能静默失败。

## 做法/步骤

我按四段拆：采集、清洗、摘要、投递。OpenClaw 负责调度和胶水层，MCP/插件只提供具体能力。

### 1. 采集

用 RSSHub 或站点原生 feed，统一通过一个 `read-rss` MCP 工具抓取。每条 item 生成稳定 ID：优先使用 `guid`，没有则用 `link + published` 做 SHA-256。原始 item 写入本地 SQLite 或 JSONL，状态字段包含：`fetched_at`、`cleaned_at`、`summarized_at`、`delivered_at`。

### 2. 清洗

先把 `content:encoded`、`summary`、`description` 合并，再做 HTML 剥离、去除脚本/style、折叠空白，最后截断到 6000–9000 字符。对没有全文的 feed，只根据标题和摘要做短摘要；不要在抓取阶段自动访问原文，容易引入登录墙和反爬。

### 3. 摘要

提示词固定三件事：

- 用中文给 3–5 条要点；
- 标注“值得读 / 一般 / 可跳过”；
- 如果原文信息不足，明确输出“仅依据摘要，可能不完整”。

输出 JSON，不要 markdown 散文。批量处理时每批 8–12 条，`temperature` 控制在 0.2–0.3。

### 4. 投递

生成 Markdown digest，按源分组或按优先级排序，推送到 Webhook、邮件或 Obsidian。保留原文链接和 item id，方便回溯。

一个最小配置示例：

```yaml
pipeline:
  fetch:
    cron: "*/30 * * * *"
    max_items_per_feed: 10
  clean:
    max_chars: 8000
  summarize:
    batch_size: 10
    temperature: 0.2
    output: json
  deliver:
    channel: "openclaw_webhook"
```

## 踩坑点

- **不要把“最近 30 条”当增量**：feed 可能返回静态列表，必须用持久化 item id 做差集，否则会重复摘要。
- **时区**：`published` 可能是 GMT，本地投递要按 UTC+8 转换；cron 也要显式指定时区，否则早上会看到昨天内容。
- **HTML 剥离后可能只剩一句话**：很多站点 RSS 只给摘要，清洗前先判断 `content` 字段真实长度，不足 300 字就不要做长篇摘要。
- **LLM 输出格式漂移**：即使要求 JSON，也要加解析容错：先提取首个 `{ ... }` 块，失败则退回纯文本摘要，不要让整批失败。
- **限频与 token**：RSS 抓取可以频繁，但摘要要合并。建议 fetch 30 分钟一次，summarize 2–4 小时一次，按“新增未摘要数”触发，而不是每条实时摘要。
- **源挂掉静默失败**：为每个 feed 记录 `last_success_at`，超过 24 小时未成功就输出告警；不要用 try/except 吞掉所有异常。

## 可复用建议

- 将“采集 / 清洗 / 摘要 / 投递”拆成独立工具或 MCP server，避免一个 Prompt 里做所有事。
- 用状态表驱动：`fetched=0/1`、`summarized=0/1` 比在内存里传对象可靠，重跑也幂等。
- 对重要源维护白名单，摘要后打标签，例如 `#must-read`、`#skip`，用规则过滤低信息量条目。
- 每周人工抽检 5–10 条摘要，发现模型开始“偷懒”就更新 few-shot 示例，而不是不断加长提示词。
- 原始 feed 文件和清洗后文本至少保留 7 天，便于排查摘要错误来源。

## 总结

RSS + AI 摘要的关键不是“接入大模型”，而是把管线做成可重跑、可观测、可降级。采集层保证不丢，清洗层保证不脏，摘要层用结构化输出控制质量，投递层只发真正有价值的信息。这样即使某个源或模型临时出问题，也不会让整条信息流变成噪音。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/99e0473baa8a863a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/2ebeb9b68ea7d455.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/b252fdf4eabfd7e6.png)

