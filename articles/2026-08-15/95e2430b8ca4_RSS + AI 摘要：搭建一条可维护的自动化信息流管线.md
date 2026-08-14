---
title: RSS + AI 摘要：搭建一条可维护的自动化信息流管线
feedId: 33172
source: 综合讨论
publishedAt: 2026-08-15
---

## 背景

RSS 源越来越多，真正值得读的内容却很少。与其让未经过滤的全文占用注意力，不如在信息进入阅读列表前，先做一次摘要和过滤。本文面向已经在用 OpenClaw、Agent、MCP 或插件做自动化的用户，重点不是“接入大模型”，而是把整条管线做稳定：采集、清洗、去重、摘要、投递。

## 问题

直接把 RSS 全文丢给模型，会遇到几个工程问题：

- 条目重复：同一篇文章可能在多个源出现，或源本身重复更新；
- HTML 标签、Cookie 弹窗、导航噪音进入模型；
- 模型输出不稳定，摘要格式每次都不一样；
- 重复摘要浪费 token 和成本；
- 投递渠道分散，错误信息难以追溯。

所以需要一条可重复、可回滚的管线，而不是一次性脚本。

## 做法：五步管线

### 1. 采集与标准化

用 `feedparser`（Python）或 `rss-parser`（Node.js）读取 RSS。统一字段：

```text
id / link / title / summary / published / author
```

RSS 2.0 和 Atom 的字段差异较大，尤其时间格式。建议全部转成 UTC 时间戳。`published` 缺失时，用抓取时间兜底，但要标记为 `fallback_time=1`，后续方便排查。

### 2. 去重

以 `link` 做主键，hash 后写入 SQLite。没有 `link` 时，用 `title + published` 做备用键。不要只用标题，因为很多源会修改标题或做 A/B 测试。

```python
key = sha1(entry.link or f"{entry.title}:{entry.published}")
if db.exists(key):
    continue
```

### 3. 正文提取与清洗

很多 RSS 只给摘要，需要抓原文。用 `trafilatura` 或 `readability` 提取正文，然后清洗：

- 移除 `script`、`style`、导航、页脚；
- 只保留 `p`、`li`、`blockquote`；
- 限制长度在 4000–6000 字符，避免超出模型上下文。

不要直接把网页 HTML 扔给模型，脏数据是摘要质量差的主要原因。

### 4. AI 摘要

使用 OpenAI-compatible API。建议强制 JSON 输出：

```text
summary: 3句话以内
key_points: 3-5个要点
tags: 2-4个标签
importance: 0-3
```

`temperature` 设低一些，0.2 左右。失败时回退到首段截断，不要阻塞整条管线。

### 5. 投递与调度

投递方式可以邮件、Telegram、Webhook，或写进 OpenClaw 的插件/MCP 工具。调度用 cron 每 15–30 分钟跑一轮。如果做常驻进程，注意不要让抓取阻塞 Agent 主流程。

如果你在 OpenClaw 里跑 Agent，可以把这条管线拆成 MCP tools：

```text
fetch_rss(source)
check_duplicate(item)
extract_content(url)
summarize(text)
deliver(item)
```

这样 Agent 可以调用，脚本也可以独立运行。

## 踩坑点

- **RSS 源编码和时间解析**：统一 UTC，时间解析失败时不要丢弃条目。
- **guid 不等于 permalink**：有些源用 guid 做内部 ID，不能直接当去重键。
- **正文清洗不彻底**：Cookie 弹窗和广告会进入模型，影响摘要。
- **模型输出不稳定**：强制 JSON 或用 function calling；失败重试，再失败回退。
- **重复摘要浪费成本**：只对新增条目做摘要，摘要结果缓存起来。
- **反爬与限流**：加 UA、重试间隔，RSS 不可达时保留原始条目，下一轮再试。
- **内容过长被截断**：截断可能切掉结论，必要时让模型基于前 N 字生成“要点”，而不是强行全文覆盖。

## 可复用建议

- 用 `link` 做幂等，不要依赖标题或时间；
- 摘要结果存 JSON，便于 rerun 和调试；
- 抽象三个接口：`Source`、`Normalizer`、`Sink`，源和投递方式可以替换；
- 先跑通 3 个源，再考虑并发和队列；
- 保留原始条目和摘要版本，方便回滚 prompt；
- 监控每轮新增量、失败源、模型延迟，能快速定位是采集问题还是模型问题。

## 总结

自动化信息流管线的价值不在于“接入 AI”，而在于可重复、可回滚、可替换模型。先把去重和清洗做好，AI 摘要只是最后一环。不要在模型层面过度设计，先把脏数据挡在管线外面。

---

