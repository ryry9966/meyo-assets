---
title: RSS + AI 摘要：搭建一条可维护的自动化信息流管线
feedId: 34688
source: 综合讨论
publishedAt: 2026-08-25
---

> 适合已经会写一点脚本、正在用 OpenClaw/Agent/MCP 做自动化的读者。目标不是“做一个酷产品”，而是有一条能长期跑、可排查、不制造信息噪音的管线。

## 背景

RSS 仍然是比较稳定的信息源接入方式，但源一多，阅读成本会迅速上升。全文读完不现实，只看标题又容易漏掉关键内容。AI 摘要可以承担压缩和初筛工作，但直接把 RSS 丢给模型，通常会遇到重复处理、格式漂移、上下文过长、推送失控等问题。

这篇文章记录一条可落地的 RSS + AI 摘要管线，重点是稳定性，而不是模型效果。

## 问题

实际跑起来后，常见问题有这么几类：

- 条目重复抓取，导致重复摘要、重复推送。
- 很多 feed 只提供摘要，没有全文，模型拿不到足够上下文。
- LLM 输出结构不稳定，prompt 一改，下游解析就崩。
- 推送频率失控，最后从“信息助理”变成“噪音源”。
- 源失效、时区混乱、URL 带跟踪参数，都会破坏去重逻辑。

所以这条管线不能只是“抓下来 -> 问模型 -> 发出去”。

## 做法

整体结构如下：

```text
RSS 源 -> 抓取/标准化 -> 去重/过滤 -> 正文抽取 -> AI 摘要 -> 结构化输出 -> 推送/落盘
```

### 1. 统一 RSS 入口

建议自建 FreshRSS 或 Miniflux，必要时用 RSSHub 把非 RSS 源转成 RSS。统一入口的好处是，下游只需要处理一种字段结构。

每条目只保留这些字段：

```text
feed_id, title, link, published, content, summary
```

给每个源打上分类和优先级标签，例如 `tech/high`、`news/low`。后续摘要和推送策略都可以按标签区分。

### 2. 去重与状态管理

用 SQLite 存条目指纹：

```text
entry_hash = sha1(feed_id + link)
```

插入时使用 `INSERT OR IGNORE`，保证同一条目只处理一次。表里至少保留这些字段：

```text
entry_hash, first_seen_at, summarized_at, summary_model, summary_hash
```

不要拿发布时间做唯一键，很多站点会修改发布时间。URL 也需要规范化：去掉 `utm_*` 参数、排序 query、去掉 fragment。

### 3. 正文抽取

优先用 trafilatura 或 readability 抽取全文。如果抽取结果少于 200 字符，再回退到 RSS 自带的 summary。

清洗时去掉 script/style、多余换行，并截断到模型可接受范围，比如 4000-6000 字符。不要无脑把整页 HTML 塞给模型。

### 4. AI 摘要

摘要输出建议固定结构，例如：

```json
{
  "score": 1,
  "relevance": "high",
  "summary": "不超过120字",
  "action_items": [],
  "tags": []
}
```

能开 JSON mode 或 function calling 就尽量开，不要依赖自然语言解析。批量处理时每批放 5-10 条，控制总 token。prompt 里明确要求“不要评论、不要扩写、不要给无关建议”。

### 5. 推送与落盘

先写 Markdown 或 JSON 到本地文件，再推送。推送渠道可以用 ntfy、Telegram、邮件，或者接入 OpenClaw 的 agent action。

不建议把推送逻辑写死在摘要脚本里。更合理的做法是：摘要脚本只负责产出结构化结果，由调度层或 agent 决定是否推送、推到哪里。

## 踩坑点

- **重复推送**：通常不是 hash 算法问题，而是 link 带 utm 参数或 query 顺序不同。URL 规范化要放在 hash 之前。
- **模型输出漂移**：即使开了 JSON mode，也可能漏字段。加一层 schema 校验，失败重试一次，再失败就标记为 dead letter。
- **全文抽取失败**：某些站点反爬或需要 JS 渲染。对这类源降级为只读摘要，不要硬刚。
- **token 费用**：逐条调用模型很费。批量输入、限制每日最大处理条数，超限自动延后。
- **时区问题**：`published` 字段经常不带时区。比较时统一转成 UTC，不要用本地时间直接比。
- **源失效**：连续多次抓取失败就自动禁用该源，并推送一次告警，而不是静默失败。

## 可复用建议

- 源列表、摘要 prompt、推送渠道全部配置化，改配置不要改代码。
- 摘要结果做缓存：同一 `entry_hash` 只摘要一次，除非摘要版本或模型变更。
- 记录每次运行处理条数、失败条数、耗时、token 消耗，方便定位问题。
- 新源先 dry-run，先用 5 个以内源跑一周，再逐步扩展。
- 把 AI 摘要当过滤器，不替代原文阅读。重要条目必须保留原文链接。
- 如果已经在用 OpenClaw/MCP，可以把“抓取-摘要”封装成 MCP server，暴露 `fetch_unread`、`summarize_batch`、`push_digest` 三个 tool。这样换模型、换推送渠道，不会影响核心管线。

## 总结

这条管线最花时间的不是 AI，而是去重、抽取、状态管理和失败恢复。模型只负责压缩和提取，稳定性由工程层保证。

先跑通一个最小闭环：

```text
10 个源 -> SQLite 去重 -> 正文抽取 -> 批量摘要 -> 本地 Markdown -> 推送
```

之后再考虑分类、向量检索、个性化评分。不要一上来就做复杂架构，先让一条简单管线稳定跑一周。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/9fed0fd330420a1b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/3aeb61bc5def8e6f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/f5d54b92f01b651c.png)

