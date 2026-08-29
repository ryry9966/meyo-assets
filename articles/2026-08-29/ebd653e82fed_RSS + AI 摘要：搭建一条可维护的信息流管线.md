---
title: RSS + AI 摘要：搭建一条可维护的信息流管线
feedId: 35231
source: 综合讨论
publishedAt: 2026-08-29
---

## 背景

我平时维护十几个 RSS 源，覆盖安全公告、开源项目发版、技术博客和少数行业新闻。每天新增条目 100+，如果逐条打开，时间完全不够。最早尝试把所有内容直接丢给大模型做摘要，结果成本高、输出不稳定，还经常因为单条超长内容把任务卡死。

## 问题

这里要解决的不是“AI 会不会摘要”，而是四个工程问题：多源格式不统一、重复条目和更新条目难判断、LLM 输出结构不稳定、调度失败后状态容易乱。如果把抓取、清洗、去重、摘要、推送全塞进一个脚本，很快会变成不可维护的定时炸弹。

## 做法/步骤

我把管线拆成五层，每层只做一件事。

### 1. 抓取入库

使用 feedparser 统一接入 RSS/Atom。对每条目生成稳定 id：`entry.get("id") or entry.link`。入库前先查重，表结构如下：

```text
items(id, feed, title, link, published, summary, content, raw_hash, status, created_at)
```

增量判断不要只依赖 etag 和 Last-Modified，很多站点不返回。实际做法是二者作为加速条件，guid 或 link 哈希作为最终去重依据。请求时带上 User-Agent，并保留原始发布时间。

### 2. 清洗正文

用 trafilatura 或 readability 从原文抽取正文。失败时退回到 feed summary 的前 500 字。标题和内容先过一次规则过滤器，把转发、抽奖、广告、招聘等低信息密度内容直接标记为 noise。正文截断到 3500-5000 字符，保留开头和每段首句，避免单条过长撑爆上下文。

### 3. LLM 摘要

提示词要求严格返回 JSON，例如：

```json
{
  "title_zh": "中文标题",
  "summary": "120字以内摘要",
  "tags": ["安全", "发版"],
  "importance": 1,
  "action_items": [],
  "is_noise": false
}
```

使用 JSON mode 或 function calling，而不是让模型输出“JSON 风格文本”。提示词里明确“只基于原文，不要扩写，不要补背景”。对 `is_noise=true` 的条目直接丢弃，不再进入后续流程。

### 4. 推送

只推送 `importance>=3` 的条目到 Telegram 或邮件。同一源单日超过 3 条时合并成一条日报，降低打扰。每日固定生成 Markdown 摘要汇总，方便回溯。

### 5. 调度与状态

用 cron 或 systemd timer 每 2 小时跑一次。处理前把状态置为 `processing`，成功后置 `done`，失败回 `queued`；连续失败 3 次进入 `dead_letter`。日志记录每个源耗时、抓取数量、摘要失败率。

## 接入 OpenClaw/MCP

如果环境支持 MCP，可以将摘要结果封装成工具，例如 `list_items({feed, since, min_importance})`。OpenClaw 或 Agent 在对话中按需查询最近重要条目，而不是一次把全部摘要塞进上下文。这样既保持信息流完整，又不会占用过多 token。

## 踩坑点

- **etag 不可靠**：多数源不返回或返回格式不一致，必须用 guid/link 哈希兜底。
- **输出 JSON 失败率比想象高**：不要靠正则去抽字段，建议走 JSON mode，失败后重试一次，再失败降级为纯文本摘要。
- **公共 RSSHub 有限流**：请求间隔至少 3-5 秒，本地做缓存，或自建实例。
- **超长正文容易打死小模型**：先截断再摘要，必要时做分段摘要后合并。
- **时区混乱**：全部存 UTC，展示层再转本地时区。
- **不要把 LLM 当过滤器**：规则过滤优先，能省大量 token 和误判。

## 可复用建议

状态表保持最小化：`id`、`feed`、`published`、`raw_hash`、`status` 即可。对重复 content 做 hash 判断更新。摘要模板固定字段，避免频繁改动破坏下游解析。先跑通 2-3 个源，稳定两天再接更多源。最后把失败队列做好，比加更多功能重要。

## 总结

RSS + AI 摘要的关键不在“调模型”，而在把抓取、清洗、状态管理和失败恢复做扎实。AI 只是压缩器，真正决定管线能跑多久的是工程稳定性。把摘要结果暴露给 MCP 后，这条管线就从“推送给我看”变成“Agent 可以按需查询的信息基础设施”。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/c63bd160453c0124.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/ed8a1c2222a19061.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/019f4ba29fe2b3c4.png)

