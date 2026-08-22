---
title: RSS + AI 摘要：搭建一条能落地的自动化信息流管线
feedId: 34182
source: 综合讨论
publishedAt: 2026-08-22
---

## 背景

信息源越多，阅读负担越重。RSS 解决了“去哪看”，但没解决“看不过来”。把 AI 摘要接进 RSS，想法不新，真正麻烦的是：源不稳定、正文提取失败、LLM 输出不可控、重复内容、token 成本。折腾一圈后会发现，这不是“调一个 prompt”的问题，而是一条需要工程化的小管线。

## 问题边界

不要期望 AI 自动替你读懂一切。合理的边界是：

- RSS 负责稳定抓取和增量更新；
- 正文提取负责给模型干净的输入；
- AI 只做摘要、打标和相关性判断；
- 推送给你时保留原文链接，必要时再回看。

如果你是 Agent/MCP 用户，也建议把 AI 放在边界清晰的位置，而不是让 Agent 直接读网页、自己爬取、自己总结。链路越长，排障越难。

## 做法/步骤

我目前使用的是一条轻量管线，可根据自己的环境替换组件：

1. **配置源**：用 YAML 维护源列表，包含 `name`、`url`、`category`、`priority`、`min_interval_min`。
2. **增量抓取**：`feedparser` 解析 RSS/Atom，保存每个源的 `ETag` 和 `Last-Modified`，只处理新增条目。
3. **正文提取**：对每篇条目用 `trafilatura` 提取正文；失败时降级为 `description`，再失败只保留标题。抓取时设置真实 User-Agent，控制并发。
4. **清洗与截断**：移除脚本/导航文本；正文超过 6000 字符先截断，长文单独走分块摘要，避免 token 爆炸。
5. **LLM 摘要**：通过 API 批量处理，prompt 固定输出 JSON，包含 `summary_3_sentences`、`key_points`、`relevance_tags`、`credibility_signal`。用 `response_format` 约束 JSON，并做解析重试。
6. **去重**：用 URL hash + 标题/正文 SimHash 去重，避免不同源转载同一内容。
7. **输出与推送**：生成 Markdown digest，推送到飞书/Telegram/邮件；同时可以输出一个新的 RSS feed 供订阅。
8. **调度**：cron 每 30–60 分钟跑一次；源级别限流。

如果你在用 OpenClaw 或类似 Agent 运行时，建议把上述步骤封装成三个 MCP 工具：`fetch_rss`、`summarize_item`、`publish_digest`。Agent 只做调度和异常处理，不直接处理网页内容。

## 踩坑点

- **正文提取失败率比想象高**：很多站点反爬或页面结构复杂。必须有降级链路：正文 → description → 标题。同时记录失败原因，方便优化。
- **LLM 输出不稳定**：即使要求 JSON，偶尔也会加解释文本。处理时先提取第一个 `{` 到最后一个 `}`，再解析；解析失败重试一次，仍失败就丢弃并记日志。
- **长文成本失控**：全文直接塞给模型，每天几百条会非常贵。先按字数和关键词过滤，只对高优先级源做全文摘要。
- **重复内容**：不同源 URL 不同、标题略微不同，单靠 URL 去重不够。加内容指纹更可靠。
- **频率与封禁**：不要高频请求同一域名。保存源级最小间隔，失败后指数退避，尊重 ETag。
- **状态丢失**：抓取状态、摘要结果、去重指纹都要落盘。否则重跑会重复推送。

## 可复用建议

- 配置与代码分离，源列表不要硬编码。
- 摘要 prompt 加版本号，例如 `prompt_version: v1.2`，方便回测和对比效果。
- 输出保留原始链接、发布时间、源名、指纹和摘要版本，便于追溯。
- 提供 `--dry-run`，先看会推送什么再实际发送。
- 监控每天的成功/失败/重复/耗时；失败率突然升高通常意味着源页面改版或反爬升级。
- 从 5 个源开始跑两周，再逐步加量。先稳定，后扩展。

## 总结

这条管线不追求“全自动读懂一切”，而是把 RSS、正文提取、AI 摘要、去重和推送拆成可替换的模块。AI 只做它擅长的部分，工程侧负责可靠性和成本控制。实际跑下来，最重要的不是 prompt 写得多好，而是失败降级和状态管理是否扎实。把这些做好，后面无论是接 OpenClaw、MCP 还是其他 Agent 框架，都会顺畅很多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/dd15f386ba51480e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/2b0a798e8cb7ecfa.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/8b437b9a0cbd937f.png)

