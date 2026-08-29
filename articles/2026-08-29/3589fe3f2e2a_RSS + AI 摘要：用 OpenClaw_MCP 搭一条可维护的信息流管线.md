---
title: RSS + AI 摘要：用 OpenClaw/MCP 搭一条可维护的信息流管线
feedId: 35163
source: 综合讨论
publishedAt: 2026-08-29
---

# RSS + AI 摘要：用 OpenClaw/MCP 搭一条可维护的信息流管线

## 背景

RSS 订阅源超过十几个后，问题会从“看不到更新”变成“看不完更新”。很多条目是转载、低密度长文、标题党或无关更新。如果每条都直接丢给 AI 做摘要，通常只会得到两类结果：正确但无用的摘要，或者推送过频变成新的信息垃圾。所以这条管线更适合设计成**过滤优先、摘要其次、推送克制**，而不是给 RSS 阅读器加一个 AI 按钮。

## 问题

能长期运行的管线需要解决四件事：增量抓取、去重清洗、摘要质量、推送节奏。这些不处理，AI 摘要只会放大噪声。

## 做法

1. **先定义输出结构**。不建议模型输出散文摘要，用 JSON 字段更稳定：`one_line_summary`、`key_points`、`action_items`、`worth_reading`、`confidence`。提示词里明确要求不重复标题、不扩写背景、不猜测原文没有的信息。

2. **抓取与去重**。用脚本或 MCP 的 RSS/HTTP fetch 抓取，规范化 `link`、`guid`、HTML 实体、相对链接；用 link/guid 做 hash 去重，存 SQLite/JSON 记录 `etag`、`last-modified`、已处理 hash。状态不要只依赖 agent 的 memory，memory 适合记偏好，不适合当抓取游标。

3. **规则过滤先行**。在 AI 摘要前，先按标题黑名单、空正文、低优先级源、付费墙或纯视频内容过滤。实践上能砍掉一半以上条目，模型只处理高价值候选集，成本与噪声都会下降。

4. **AI 摘要与校验**。正文截断到 1500-3000 tokens，只保留纯文本；输出 JSON，温度设在 0.1-0.3；解析失败重试一次，再失败就标记跳过。OpenClaw/Agent 可以跑固定任务，MCP 工具尽量只读，避免给 agent 太大写权限。

5. **聚合推送与归档**。不要每条实时推送。按 6/12 小时聚合为 Markdown digest，包含一句话摘要、关键点、是否值得精读、原文链接。原始数据写 JSONL，方便回溯和重新处理。

## 踩坑点

- RSS 格式脏：相对链接、CDATA、未转义 HTML、无 guid，必须先 normalize。
- 去重只按标题会误合并或重复，优先用 link hash 或 guid。
- 模型输出飘：不用 JSON schema 时容易生成“本文介绍了……”这类无意义内容。强制 JSON 并加重试。
- 正文混入导航、相关推荐、评论区，会浪费 token 且干扰摘要，抓取时要剥离。
- 推送过频会让用户直接屏蔽消息，聚合 digest 更可持续。
- 抓取游标丢失会导致重复摘要，失败重试必须幂等。

## 可复用建议

1. 先跑 3 个源稳定一周，再接更多源，不要一上来就接几十个订阅。
2. 抓取与摘要解耦，模型或 MCP 服务不可用时，抓取过滤仍能运行。
3. 保留原文链接和原始条目，摘要出错时还能回查。
4. 提供 dry-run 模式，先输出本地 Markdown 观察摘要质量。
5. 提示词版本化，方便对比调整效果。
6. MCP 工具尽量只读，控制权限范围。

## 总结

RSS + AI 摘要的收益不在“自动读完所有内容”，而在把信噪比提高。工程重点不是模型强度，而是去重、状态管理、结构化输出和推送节奏。OpenClaw/MCP 适合做编排，插件生态能省不少胶水代码，但数据清洗和过滤规则才是长期可维护的关键。先把管线跑小、跑稳，比一开始追求全自动更实际。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/3b1d519b0e5eeeb3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/ed0e56f864f1e655.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/9cda1fc6d8e45938.png)

