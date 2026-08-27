---
title: RSS + AI 摘要：搭建低噪声自动化信息流管线
feedId: 34972
source: 综合讨论
publishedAt: 2026-08-28
---

# RSS + AI 摘要：搭建低噪声自动化信息流管线

## 背景

RSS 没有死，只是从大众订阅退回到工程信息源。技术博客、安全公告、产品 changelog 仍然适合用 RSS 跟踪。问题不在“能不能抓到”，而在量上来以后阅读带宽不够。把“抓取—清洗—AI 摘要—过滤—推送”做成自动化管线后，每天只需要扫一遍摘要，再决定是否打开原文。这个管线在 OpenClaw/Agent 体系里适合作为信息采集前端，而不是让 Agent 每次都临时抓网页。

## 问题

直接做 `cron + feedparser + GPT` 脚本，一开始能跑通，但很快会暴露几类问题：

1. 源质量不一：有的 feed 只有摘要，有的正文 HTML 噪声很大。
2. 重复推送：guid 不稳定或缺失，只按标题判重会漏，也会重复。
3. 摘要输出不稳定：开放式提示会让模型写小作文，或者漏掉关键点，下游不好消费。
4. 单点失败：一个源超时，整个任务挂掉，后面的源都受影响。

## 做法/步骤

我把管线拆成五层：抓取、清洗、摘要、过滤、推送，层与层之间用 SQLite 保存状态。

**1. 抓取与去重**

用 Python `feedparser` 读源，或者走 Miniflux API。对每条 entry 选一个稳定字段生成 `entry_hash`，优先 `id`/`guid`，没有则用 `link`。SQLite 里记录 `feed_id, entry_hash, title, link, published, status`。先查是否见过，见过就跳过。某些源会重发修订版本，所以保留 `updated` 字段一起比对。

**2. 清洗正文**

不要让模型直接吃 HTML。用 `trafilatura` 或 BeautifulSoup 提取正文，截断到 3 万字符以内；拿不到正文就回退到 `summary`。清洗后组装成 `feed_title + title + text` 的结构，保留链接、作者、发布时间。

**3. AI 摘要**

使用 OpenAI-compatible API，温度设低，输出固定 JSON：

- 一句话摘要
- 关键点三条
- 相关度 0-5
- 是否值得深读及理由

批量处理，一次 10-20 条，避免逐条调用。解析时用 `json_repair` 兜底，因为模型偶尔会包代码块或输出尾逗号。API 失败做指数退避，不要立刻中断整批。

**4. 过滤与动作**

按相关度阈值过滤，例如 3 分以上推送，低于阈值只落库。状态可以设计成 `new / summarized / pushed / low / manual_review`。管线的作用是降噪，不是替你删信息。

**5. 推送与下游消费**

推送到 Telegram、webhook 或写入 Obsidian/Notion，保留原文链接和摘要 hash。后续 OpenClaw 里的 Agent 可以通过 MCP 工具查 digest 库，而不是重新抓网页。

伪代码：

```python
entry_hash = stable_hash(entry.get("id") or entry.link)
if db.seen(entry_hash):
    return
text = extract_text(entry)
items = batch_summarize([...])
for item in items:
    if item.score >= 3:
        push(item)
    db.save(entry, item, status="pushed")
```

## 踩坑点

- 不要用 title 判重，feed 标题会变，id/link 的 hash 更可靠。
- 摘要提示词不要写“帮我总结这篇文章”，要求 JSON 输出才能接入自动化。
- 不要逐条调 API，批次化能显著降低延迟和花费。
- 抓取频率控制在 15-30 分钟一次，既够用，也不容易触发限流。
- 每个源独立包裹异常，单个 feed 的编码错误、证书错误、超时都不该拖垮整条任务。
- 有版权或授权限制的源，只做个人摘要，不要公开分发全文。

## 可复用建议

把源列表、提示词、阈值全部放进配置文件，不要硬编码。保留原文 hash 和摘要 hash，方便重放、回填和排查。建立一个 `manual_review` 队列，低分但不确定的条目可以人工扫一眼，避免漏掉冷门但重要的更新。与 OpenClaw/Agent 结合时，建议边界划清：管线负责“读什么、何时读、怎么摘要”，Agent 负责“基于摘要做什么”。可以把摘要库暴露成 MCP tool，让 Agent 查询后再执行写笔记、发提醒或触发后续任务。

## 总结

RSS + AI 摘要的自动化管线成本不高，真正决定稳定性的是工程细节：稳定去重、正文清洗、结构化输出、失败重试、错误隔离。把这些做好，它就能从一个一次性脚本，变成长期可用的信息前端。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/993217958f63f2c1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/e1451e96d42429e7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/7fb4bd90997cf048.png)

