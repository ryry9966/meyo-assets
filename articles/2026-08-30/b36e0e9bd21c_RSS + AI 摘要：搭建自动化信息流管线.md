---
title: RSS + AI 摘要：搭建自动化信息流管线
feedId: 35399
source: 综合讨论
publishedAt: 2026-08-30
---

# RSS + AI 摘要：搭建自动化信息流管线

## 背景

RSS 重新成为个人知识管理里较可靠的输入边界：源可控、没有推荐算法、可以离线抓取。但当订阅数超过几十个，每天新增上百条，读不过来。逐条人工过滤很消耗注意力；如果对每条全文直接调用 AI 摘要，又容易成本高、结果噪。

## 问题

- 原始 feed 质量参差：标题党、重复转载、摘要截断、时区错乱。
- 全量 AI 摘要会产生大量“正确的废话”，反而增加阅读负担。
- 抓取、摘要、推送耦合在一起时，某一步失败会导致整条管线不可用。

所以更适合拆成三阶段：抓取清洗 -> 规则过滤 + AI 摘要 -> 投递与沉淀。

## 做法/步骤

### 1. 源与抓取

用 Miniflux 或 RSSHub 生成规范 feed；抓取端可以用 Python `feedparser` 或 Miniflux API。定时任务每 30 分钟跑一次，保存 raw JSON 到 SQLite 或对象存储。关键字段建议保留：`feed_id`、`url`、`title`、`published`、`content_hash`。

### 2. 清洗与去重

用 URL 规范化 + SHA-1 `content_hash` 去重；标题归一化去掉站点前缀、日期噪声。正文用 `trafilatura` 抽取，避免原始 HTML/CSS/脚本污染摘要输入。只保留近 24~48 小时新条目，超过窗口的标记为 `stale`。

### 3. 规则过滤 + AI 摘要

先做一层无成本过滤：标题正则黑名单、关键词白名单、来源优先级、长度阈值。对进入 AI 阶段的条目做 batch，通常 10~20 条一批。示例 prompt：

> 你是信息编辑。对每条 RSS 条目输出 JSON：`score` 0-10，`keep` 布尔值，`summary` 不超过 60 字，`tags[]`。低信息密度、纯推广、重复内容直接 `keep=false`。

使用 `temperature=0` 或 `0.1`，要求严格 JSON。失败重试一次，再失败就原样投递标题，不阻塞管线。

### 4. 投递与沉淀

生成 Markdown digest，按来源分组，每条给 2~3 句摘要、原文链接和 score。通过 Webhook 推到 Telegram、飞书或 ntfy，同时写回数据库。

如果使用 OpenClaw 做编排，可以把 `summarize_feed`、`list_unread`、`deliver_digest` 封装成 MCP tools，Agent 可以按需查询和二次过滤，例如：“今天有哪些安全相关的更新？”

### 5. 调度与可观测

`cron` 或 systemd timer 已经够用；复杂编排可以用 n8n。每次任务写入 `run_log`：抓取条数、去重条数、AI 成功/失败、推送状态。失败可重放且要保持幂等。

## 踩坑点

- `feedparser` 对部分 feed 的 `published` 字段时区不一致，统一转 UTC 再比较。
- 大模型 JSON 输出不稳定，不要让它输出 Markdown 表格再解析；用 JSON Schema + retry。
- 全文抽取失败时，不要用 feed 自带 `summary` 直接替代，容易产生垃圾输入；标记 `extraction_failed`，仅投递标题链接。
- 成本控制：不要每条单独调用，batch 能降低调用次数；大量来源可以先按 score 截断。
- 推送频率不要过密，否则容易被 IM 风控，也容易变成另一种信息噪声。

## 可复用建议

- 配置外置：`sources.yaml` 描述 feed、优先级、标签、是否启用全文抽取。
- 阶段解耦：`fetch` / `filter_summarize` / `deliver` 三个独立脚本或 task，任一步可以单独重跑。
- 保留原文链接：摘要只负责“判断是否值得点开”，不替代深读。
- 从 10~20 个高质量源开始，稳定后再扩量。
- 对重要源设置强制投递，即使 AI 判低分也要出现，避免误杀。

## 总结

这个管线不追求“更多信息”，而是把 RSS 从待办列表变成可扫描的简报。抓取、清洗、过滤、摘要、投递分层后，每个环节都能独立替换：换模型、换推送通道、换存储都不影响整体。控制好批处理与重试，基本可以稳定无人值守运行。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/a02f5dbe4830e5e0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/a13f3725331a0ffd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/5991ae423d009a76.png)

