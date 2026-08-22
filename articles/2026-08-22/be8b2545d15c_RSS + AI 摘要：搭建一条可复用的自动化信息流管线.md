---
title: RSS + AI 摘要：搭建一条可复用的自动化信息流管线
feedId: 34131
source: 综合讨论
publishedAt: 2026-08-22
---

## 背景

RSS 让信息源回到可控状态，但随着订阅数增加，很容易从“读不完”变成“未读焦虑”。每天几百条更新里，真正和当前工作、技术方向相关的可能只有几条。目标不是读得更多，而是让系统先做一轮筛选和压缩，我只处理值得看的内容。

这条管线适合已经使用 OpenClaw、Agent、MCP 或自动化工具的用户：它不绑定某个阅读器，而是把 RSS 抓取、AI 摘要、去重、推送串成一个可调度、可复用的数据流。

## 问题拆解

自动化信息流需要解决四件事：

1. 稳定抓取 RSS，并保留已读/未读状态；
2. 清洗 HTML，拿到干净的正文或摘要；
3. 用 LLM 做摘要、打分和过滤；
4. 去重后推送到通知渠道，而不是全量倾倒。

如果只是手动在 RSS 客户端里看，这些都不是问题。但一旦要让 Agent 使用，就必须把它做成无状态、可增量、可返回结构化结果的管线。

## 做法

整体架构如下：

```
RSS 源 -> Miniflux/RSSHub -> fetcher -> 清洗截断 -> LLM 摘要/打分 -> SQLite 去重 -> ntfy/Telegram
```

### 1. 抓取层

使用 Miniflux 作为 RSS 后端，它提供 REST API，能保存已读/未读状态，避免每次重复处理。没有 RSS 的站点用 RSSHub 补全。脚本只面向 Miniflux API 工作，不绑定具体阅读器。

### 2. 清洗层

拉取最近未读条目后，用 `html2text` 或 `BeautifulSoup` 去掉脚本、导航、广告等噪声。正文只保留前 3000~5000 字符，既给 LLM 足够上下文，又避免 token 浪费。

### 3. 摘要与过滤层

调用 OpenAI-compatible 的 chat completions API，要求强制输出 JSON：

```text
你是信息过滤助手。阅读以下 RSS 条目后输出 JSON：
{"worth_reading": true/false, "summary": "不超过80字", "tags": ["技术/产品/行业/其他"], "reason": "一句话理由"}
只输出 JSON，不要 markdown。
```

温度设 0.2，尽量使用 JSON mode 或 tool call。解析失败时剥离 ```json``` 和 ``` 后重试，最多 2 次。

### 4. 去重与存储

SQLite 表结构可以很简单：

```text
entries(id, feed_id, url_hash, title, summary, tags, score, created_at, pushed_at)
```

用 `sha1(normalize_url(link))` 做主键去重。同内容不同 URL 的情况，可以对标题做小写、去空白后的 simhash 辅助降重。

### 5. 推送

只推 `worth_reading=true` 且 `score>=0.7` 的条目。推送内容包含摘要、理由、原文链接。单次最多推 5 条，避免信息流变成新的骚扰源。未推送但分不低的条目可以进入稍后读表。

### 6. MCP/Agent 封装

如果在 OpenClaw、Claude Desktop 等 MCP host 中使用，可以把 fetcher 和 summarizer 封装成 MCP tools，例如：

```text
rss_summary(feed_id, limit=10)
```

调度仍然由外部 cron 触发，MCP server 只做无状态查询，不要让它常驻消费、自己管理定时任务。

## 踩坑点

- **RSS 只有标题或截断正文**：不要直接把截断文本交给 LLM。优先选择自带全文的源；没有全文时，用 RSSHub 的正文提取或 readability 兜底。
- **LLM 输出不稳定**：即使要求“只输出 JSON”，模型偶尔也会包 markdown 代码块。解析前先清理 ```json``` 和 ```，开启 `response_format=json_object` 能显著提高稳定性。
- **重复推送**：URL hash 能处理完全相同的链接，但标题近重复的内容仍会漏过。标题小写、去空白后做 simhash，阈值设 3~5 可以有效降重。
- **成本与限流**：不要全量重跑。使用 Miniflux 的 `after_entry_id` 或 `updated_at` 做增量拉取，每批 10~20 条。
- **推送疲劳**：重要条目即时推送，一般条目每日汇总一次。设置最低分比设置复杂分类更有效。

## 可复用建议

- 状态外置到 SQLite，脚本无状态，方便被 Agent、MCP、定时任务反复调用。
- 每次 LLM 响应连同原文 URL 追加写入 JSONL，方便回放、调试和重刷 prompt。
- Prompt 中明确加入“与当前关注主题无关可直接 worth_reading=false”，让过滤更贴合个人目标。
- 模型配置放环境变量，便于切换便宜模型处理短摘要，贵模型处理长文精读。
- 公共源走 RSSHub 缓存，个人订阅走 Miniflux，源和抓取职责分离。

## 总结

RSS + AI 摘要管线的核心不是“接入大模型”，而是过滤、去重、全文质量和推送频率。把它们做成无状态 API 或 MCP tools 后，可以持续作为 Agent 的信息输入层。先用最小版本跑通，再逐步加评分、汇总和多源降重，比一开始上复杂架构更实际。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/3f066c427ef129c9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/2307e4c46b17b443.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/56f450f9796861fd.png)

