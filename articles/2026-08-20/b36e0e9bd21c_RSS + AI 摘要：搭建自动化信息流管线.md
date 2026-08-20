---
title: RSS + AI 摘要：搭建自动化信息流管线
feedId: 33929
source: 综合讨论
publishedAt: 2026-08-20
---

## 背景

RSS 依然适合追踪个人博客、发布说明、产品文档、安全公告和 Newsletter。但当订阅源超过 30 个，每天会稳定产出上百条更新。原始 RSS 混杂标题党、只含摘要的条目、重复转载和低信息密度内容。直接让 Agent 读取原始 feed，会出现上下文膨胀、成本上升、抓不到重点的问题。

这篇帖子面向已经接触 OpenClaw、MCP 或 Agent 自动化的用户，讲一套可以落地的 RSS + AI 摘要管线。不追求大而全，先解决“读得完、进得去、用得上”。

## 问题拆解

一条 RSS 从源到可用，通常要经过四个问题：

1. 格式不统一：不同 feed 的字段、编码、正文完整度差异大。
2. 重复内容多：同一文章被多个源收录，或者同一来源重复推送。
3. 正文缺失：很多 RSS 只提供摘要，直接摘要容易失真。
4. 结果难结构化：如果只是让 LLM 随便总结，后续分发、检索、Agent 调用都会很痛苦。

所以管线的核心不是“调一次 LLM”，而是把脏数据先清洗到可处理状态，再让模型输出稳定结构。

## 做法与步骤

整体架构可以拆成五步：采集 → 归一化 → 过滤/去重 → AI 摘要 → 路由/分发。

### 1. 采集与入库

用 `feedparser` 或现成 RSS 服务拉取源。建议把原始数据落到 SQLite，字段至少保留：

```text
feed_title, entry_id, link, title, summary, published, raw_content_hash
```

去重键优先用 `entry_id`；没有 `entry_id` 时，用 `link` 做 hash。踩过坑的场景：某些 feed 的 `entry_id` 会变化，但 link 稳定，所以建议双键存储。

### 2. 正文抽取

如果原始 `summary` 少于 300 字，或者摘要字段明显被截断，可以尝试用 `trafilatura` 从原文链接抽取正文。设置 10 秒超时，失败就退回 RSS 摘要，不要卡死整条管线。

正文抽出来先做一次轻量清洗：去掉脚本、导航、广告残留、连续空行。这个步骤对后续摘要质量影响很大。

### 3. AI 摘要与结构化

把清洗后的正文截断到 4000–6000 字符，避免上下文浪费。Prompt 要求只返回 JSON，不要给模型自由发挥空间。推荐 schema：

```json
{
  "title_zh": "中文标题",
  "summary_zh": "不超过120字的中文摘要",
  "tags": ["rss", "ai"],
  "importance": 2,
  "actionable": false,
  "reason": "为什么重要或为什么可以忽略"
}
```

参数建议 `temperature=0.2`，强调“只输出 JSON，不要包含解释”。解析端要做 JSON 校验，失败时重试一次；再失败就保留原文链接，不阻塞后续条目。

### 4. 路由与分发

根据 `importance` 和 `actionable` 决定分发策略：

- `importance >= 2` 或 `actionable=true`：推送到 Telegram / 飞书 / 邮件。
- 所有摘要：追加到当日 Markdown digest 或本地知识库。
- 可选：将结构化结果写入 OpenClaw 可检索的 notes 或 memory，让 Agent 可以按需召回。

这样不是把所有内容都推给 Agent，而是让 Agent 需要时能查到已经摘要过的信息。

### 5. 调度

用 OpenClaw 的定时任务或 systemd timer / GitHub Actions 每小时跑一次。每次只处理上次成功时间之后的条目。首次运行不要直接 backfill 全部历史，否则 token 成本容易失控。可以加一个 `--backfill` 显式参数。

## 踩坑点

- **重复内容**：有些站点同一篇文章在不同 feed 里 link 不同。标题 + 域名 hash 可以作为兜底去重。
- **编码问题**：部分 RSS 声明 UTF-8 但实际是 GBK 或 Latin-1。`feedparser` 有时会吐乱码，需要根据 `response.encoding` 或 `chardet` 修正。
- **JSON 解析失败**：模型偶尔会输出额外文本。每条独立处理，单条失败只记录状态，不要整批中断。
- **成本控制**：先过滤低质量条目：标题黑名单、非目标语言、正文长度过短。摘要优先用便宜模型或本地模型，importance 为 1 的条目只生成一句话摘要。
- **时间窗口**：定时任务失败后，下次运行可能积压大量条目。建议限制每次最多处理 50 条，剩下的留到下一轮。

## 可复用建议

1. 输出结构保持稳定，不要频繁改 schema。下游通知、存储、Agent 检索都依赖这个结构。
2. 摘要里永远保留原始 `link` 和 `source`，否则摘要没有出处，后续无法追溯。
3. 用 `content_hash` 做二级去重，避免同一内容被重复摘要。
4. 结果存 JSONL 或 SQLite，便于后续做二次过滤、周报生成或 RAG 索引。
5. 如果已经用 MCP，可以把管线包装成一个 MCP server，暴露 `get_digest`、`search_digest`、`fetch_now` 给 OpenClaw Agent 调用。这样 Agent 从“被动接收推送”变成“主动检索信息”。

## 总结

这套管线的最小闭环是：采集 → 去重 → 正文抽取 → AI 摘要 → 本地 Markdown。先跑通这个闭环，再接通知和 Agent 工具。实际落地时，最花时间的一般不是接模型，而是处理 RSS 的脏数据和稳定 JSON 输出。把这两点控制住，后续扩展会顺很多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-20/edb93a78263ffa3e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-20/983129d2bd3b4b47.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-20/1a1367eeeb1f85f8.png)

