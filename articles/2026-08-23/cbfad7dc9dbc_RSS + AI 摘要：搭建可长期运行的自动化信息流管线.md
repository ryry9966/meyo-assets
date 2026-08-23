---
title: RSS + AI 摘要：搭建可长期运行的自动化信息流管线
feedId: 34385
source: 综合讨论
publishedAt: 2026-08-23
---

## 背景

信息源越来越多，RSS 重新成为可掌控的订阅入口。但订阅源涨到几十个以后，未读数量很容易失控。很多内容只需要知道核心结论，并不需要逐字精读；如果每篇都点开，上下文切换成本很高。

直接把文章丢给通用聊天工具做摘要，临时用没问题，但很难复用：状态不连续、格式不稳定、成本和限流不可控。更工程化的做法是把 RSS 抓取、正文提取、去重过滤、AI 摘要、通知推送串成一条自动化管线。这个管线不复杂，但边界情况很多，适合作为 OpenClaw/MCP/Agent 实践里的基础信息层。

## 问题

- **源质量差异大**：有些 RSS 只有摘要，有些全文混入大量脚本和广告；有的 link 不是原文而是跳转。
- **重复与噪音**：多个源转载同一内容，或者某个源高频更新低价值内容。
- **摘要调用不稳定**：API 超时、限流、输出格式漂移，会导致管线中断。
- **状态管理**：每次全量重新处理，成本高且容易重复推送。

## 做法/步骤

我自己的方案是 `feedparser + SQLite + Python`，减少服务依赖。整体分为五步：

1. **抓取与增量标记**  
   定时任务用 `feedparser` 拉取 RSS，SQLite 保存条目。表结构至少包含：`id`、`feed`、`link_hash`、`title`、`summary`、`status`、`created_at`、`processed_at`。用 `last_processed_at` 或 `last_entry_id` 做增量游标，不要每次从全部历史开始。

2. **正文提取**  
   对 `link` 做正文提取。Python 里 `trafilatura` 比 `newspaper3k` 更稳，返回空时回退到 RSS 的 `description`。回退内容可能带 HTML 标签，需要先清洗。

3. **去重与过滤**  
   以 `link_hash` 或 `guid` 作为主键去重。对于不同链接但内容相同的转载，可以对正文前 300 字符做哈希，或使用简单的 SimHash/MinHash。先做关键词白名单/黑名单过滤，能明显降低后续 AI 调用量。

4. **AI 摘要**  
   使用 OpenAI-compatible API，要求输出 JSON：

   ```json
   {
     "summary": "...",
     "topics": ["..."],
     "actionable": true
   }
   ```

   建议开启 JSON mode 或结构化输出，`temperature` 调低，`max_tokens` 控制在 200–400。批量处理时串行执行，失败重试 2 次，指数退避，避免限流。

5. **输出与推送**  
   摘要写入 SQLite，并生成 Markdown 文件或通过 ntfy/Bark/Telegram 推送。也可以把抓取、摘要、推送封装成 MCP 工具，让 OpenClaw 里的 Agent 按需触发，而不是完全被动定时执行。

## 踩坑点

- **RSS 源不规范**：有些源只输出摘要，去原站抓正文可能被反爬。这类源可以降级为只摘要 `description`，不要强行全文提取。
- **正文提取失败**：`trafilatura` 可能返回空，回退到 `description` 时容易残留 HTML 标签。清洗时注意去掉 `script`、`style`、广告段落。
- **AI 输出格式漂移**：即使开启了 JSON mode，也可能偶尔解析失败。解析失败时兜底为纯文本摘要，不能因为格式问题让整条管线崩溃。
- **限流与成本**：不要一拿到新条目就全部并发调用。用队列加间隔控制，单次任务设置最大条目数。先做规则判断，明显不相关内容直接跳过。
- **重复推送**：转载内容不能只靠 URL 去重。多个源可能用不同链接发同一篇文章，需要标题相似度或正文前 300 字符哈希辅助判断。
- **时间字段缺失**：有些源的 `pubDate` 为空，用抓取时间作为补充，避免增量游标判断错误。

## 可复用建议

- **模块拆分**：`fetcher`、`cleaner`、`summarizer`、`notifier` 各自独立，方便替换。
- **配置外置**：源列表用 YAML/JSON 管理，包含 `url`、`name`、`enabled`、`tags`、`max_items_per_run`。
- **状态可恢复**：失败条目进入 `retry` 状态，而不是直接丢弃。记录每次 run 的抓取数、过滤数、摘要成功/失败数。
- **与 OpenClaw/Agent 集成**：把摘要结果作为 MCP resource 或 tool 输出，让 Agent 可以按需查询“今天有哪些值得关注的安全更新”，而不是被动接收全部推送。

## 总结

RSS + AI 摘要管线的核心不是“接入 AI”，而是把脏数据、重复、限流、状态管理等边界处理做好。一个稳定的管线应该是：增量抓取、清洗去重、结构化摘要、可重试、可观测。做到这些之后，再把它暴露给 OpenClaw 或 MCP，就能从“又一个自动化脚本”变成可被 Agent 调用的信息基础设施。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/988190a12f7ff3e6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/825c039496b606fc.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/34285cc0e1e36c39.png)

