---
title: RSS + AI 摘要：搭建自动化信息流管线
feedId: 34373
source: 综合讨论
publishedAt: 2026-08-23
---

## 背景

RSS 解决了“把分散更新聚到一起”的问题，但没有解决“读不过来”。大量订阅源每天产生几十上百条更新，打开后一半是标题党或无效信息。把 AI 摘要接进 RSS 管线，可以把原文压缩成结构化要点，只保留关键事实和可行动信息。

但朴素的“抓一条、调一次 AI、推一条”很容易变成碎片管道：重复抓取、重复摘要、源一挂全停、token 消耗不可控。我们需要一条工程上可维护的自动化信息流管线。

## 问题

目标不是“能跑”，而是达到四个要求：

- 增量处理，不重复摘要同一篇文章
- 单源失败不影响全量抓取
- 成本可控，有每日预算和过滤策略
- 结果可被 OpenClaw Agent/MCP 稳定消费

## 做法

### 1. 分层架构

建议把管线拆成六层：

- 源层：RSSHub 或 Miniflux，统一生成/管理订阅源
- 抓取解析层：feedparser 或自写脚本，负责拉取和解析
- 正文提取层：trafilatura 优先，readability 兜底
- 去重状态层：SQLite 或 Redis，用 hash 指纹做唯一约束
- AI 摘要层：OpenAI 兼容 API，固定输出结构
- 输出层：Markdown、ntfy、Telegram/飞书，或 MCP 工具

### 2. 关键实现

- 定时触发：cron 或 OpenClaw scheduled task。每轮先读取状态，只处理未见过的新条目。
- 规范化去重：对 `link` 去掉 `utm_*`、`fbclid` 等参数；如果 `link` 缺失，用 `title + pubDate` 生成备用 key。对每个条目算 hash，写入 SQLite 唯一索引，避免重复摘要。
- 正文提取：很多 RSS 只给摘要，需要补全文。trafilatura 对正文识别较稳，但部分站点会返回空，此时 fallback 到 readability。设置 UA、10s 超时，避免卡死整轮。
- AI 摘要：固定 prompt，要求只提取事实、不推断、不补背景。建议输出 JSON：`{title, summary, keywords, score}`。`max_tokens` 控制在 200–350，`temperature` 设低。对有 `response_format=json_object` 的接口直接开启，减少 Markdown 返回。
- 成本控制：不无差别摘要。跳过正文小于 500 字符的短内容；超过 6000 字符先截断。每轮设置最大处理条目数，每日设置 token 预算。失败重试用指数退避，避免 rate limit 后连续打爆。

### 3. 与 OpenClaw 结合

不建议每来一条都推送，容易造成打扰。更好的方式是：

- 摘要结果写入 SQLite 或 JSONL
- 暴露 MCP 工具：`list_summaries(since, limit)`、`get_summary(id)`
- OpenClaw Agent 按需查询，而不是被动接收全部推送
- 只有命中关键词或高优先级源才即时推送，其他合并成每日 digest

这样 Agent 拥有一个稳定、可查询的信息入口，而不是被碎片推送淹没。

## 踩坑点

- **源失败隔离**：单个 RSS 解析失败不应中断整轮。按源 try/except，记录错误后继续。
- **link 重复但 guid 不同**：同一篇文章在不同源可能 guid 不同，必须做规范化，否则重复摘要。
- **正文提取失败率高**：有些站点反爬或正文标签混乱。失败时只存链接、不摘要，并记录失败率，方便后续调整。
- **AI 输出不稳定**：要求 JSON 但有时返回 Markdown 或多余解释。用 `response_format` 约束，并做后处理容错。
- **时间戳漂移导致漏抓**：有些源会更新 `pubDate`，不能只依赖时间排序。用 hash 状态做幂等最稳。
- **推送噪音**：不要每条都推，按关键词、评分或来源优先级过滤。

## 可复用建议

- 配置化：用 `sources.yaml` 定义源、优先级、关键词、每日上限。
- 先 dry-run：跑一轮看过滤统计和 token 成本估算，再正式启用。
- 保留原始抓取 JSON 和摘要结果，便于回溯和优化 prompt。
- 日志记录每个 stage 的成功率、耗时、剩余 token，做成可观测管线。
- 小规模起步：先跑 10–20 个高质量源，稳定后再扩量。

## 总结

RSS + AI 摘要的核心不是“接上 AI”，而是把抓取、去重、正文提取、摘要、推送做成一条幂等、可观测、可限流的流程。只有这样，OpenClaw Agent 才能有一个稳定可靠的信息入口，而不是被碎片化内容反复打断。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/c3b1d6587c26f51f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/df83529339306fc1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/f4710306ec7f161a.png)

