---
title: RSS + AI 摘要：从抓取到推送的自动化信息流管线实践
feedId: 34160
source: 综合讨论
publishedAt: 2026-08-22
---

## 背景

RSS 依然是个人可控信息源里最省心的一种：没有推荐算法、没有时间线焦虑，订阅源由自己决定。但问题也明显——订阅多了以后，未读列表很快变成另一种信息过载。与其每条都点开，不如让 AI 先做一层粗筛和摘要，把值得看的内容推给自己。

在 OpenClaw/Agent 体系里，这类“抓取—清洗—摘要—推送”的流程很适合做成可复用管线：前期用脚本跑通，后期封装成 MCP 工具或插件，让 Agent 按需调用。本文记录一套轻量但工程上可落地的做法。

## 问题

直接“抓 RSS 然后丢给 AI 总结”会遇到几个现实问题：

- RSS 源质量参差不齐：有的只给标题和摘要，有的带大量 HTML，有的编码混乱。
- 重复内容：同一篇文章可能被多个源转发，或者 feed 更新时重复返回旧条目。
- AI 输出不稳定：摘要时好时坏，JSON 格式容易崩。
- 推送频率难控：源一多，每小时推几十条，很快就变成骚扰。

因此需要一个管线，而不是一个 prompt。

## 做法

### 1. 采集层

建议用 `feedparser` 或 Miniflux/FreshRSS 的 API 作为统一入口。对于不提供 RSS 的站点，可以接 RSSHub 或自建 bridge，但不要把所有希望都押在单一源上。

配置化源列表，例如：

```yaml
sources:
  - name: "AI 工程"
    url: "https://example.com/feed.xml"
    tags: ["ai", "engineering"]
  - name: "安全公告"
    url: "https://example.org/rss"
    tags: ["security"]
```

### 2. 清洗与去重

抓回来后先做标准化：

- 用 `BeautifulSoup` 或 `html2text` 移除 script/style/广告节点。
- 只保留标题、链接、发布时间、作者、正文前若干字。
- 以 `entry.link` 或 `entry.id` 做唯一键，计算 hash 后写入 SQLite/Redis。不要用标题去重，标题经常被改。

### 3. AI 摘要

摘要阶段用 OpenAI-compatible API 即可，关键是 prompt 和输出约束。

建议 prompt 固定成三部分：

- 用 3 句话说明这篇文章解决了什么问题。
- 列出关键数据、结论或争议点。
- 判断是否值得精读，并给出 one-line 理由。

温度设低，例如 `0.2`。输出用 JSON schema 约束：

```json
{
  "summary": "string",
  "key_points": ["string"],
  "worth_reading": true,
  "reason": "string"
}
```

不要要求 AI 输出太长，限制在 120 字以内，摘要阶段的目标是过滤，不是替代阅读。

### 4. 存储与推送

每条原始条目和 AI 摘要都落库，保留 `raw` 和 `summary` 两个字段。这样后续想换模型重新生成摘要，不用重新抓取。

推送层做聚合窗口：每 30 分钟或 1 小时，把新摘要打包成一条消息，按 tag 分组后发到企业微信、飞书、Slack 或 ntfy。不要每条推一次。

### 5. 调度与 MCP 封装

用 cron、GitHub Actions 或 systemd timer 周期执行。流程稳定后，可以把“抓取 + 摘要”封装成 MCP server 的两个工具：

- `fetch_rss(source)`：抓取并返回清洗后的最新条目。
- `summarize_entry(url)`：对单条链接做 AI 摘要。

这样 Agent 不需要关心底层抓取细节，也能在对话中按需调用。

## 踩坑点

**RSS 只有摘要没有全文。**  
很多源只给前 100 字，AI 摘要会严重失真。可以先用 RSSHub 找全文输出，或者对重点源做 readability 提取。不要默认 feed 内容等于原文。

**HTML 清洗不彻底。**  
`<pre>`、代码块、Cookie 提示、相关推荐等噪音会污染 prompt。清洗后最好再做一步纯文本截断，控制在 1500-2000 token 以内。

**用标题或发布时间判断重复不可靠。**  
有的 feed 每次更新都会改时间戳，有的转载改了标题。优先用 `guid`，没有 `guid` 再用 `link`。如果源特别乱，可以加标题相似度兜底。

**AI 返回 JSON 不合法。**  
即使 prompt 里写了 JSON，模型仍可能输出 `“Here is the result: {...}”` 或漏逗号。一定要做 schema 校验，失败就重试一次；重试仍失败就把原文存进 `failed_queue`，不要中断整个批次。

**推送太频繁。**  
单条推送看起来及时，但源超过 15 个后会变成噪音。聚合推送 + 可配置静默时段，比实时推送更可持续。

## 可复用建议

- 先跑 CLI，再考虑接 OpenClaw/MCP。单文件脚本能快速验证摘要质量，避免一开始就被插件框架绑架。
- `raw` 和 `summary` 分开存，模型升级或 prompt 调优后可以批量重新生成。
- 源列表用 YAML/JSON 管理，新增源不要改代码。
- 每个源记录 `last_fetched` 和 `etag`，避免重复抓取。
- 失败要有队列和日志，不要静默丢数据。
- 开始时源控制在 15-20 个，跑稳一周再扩量。

## 总结

RSS + AI 摘要的价值不在于“读完所有信息”，而是把信息流从“点开才知道值不值得看”变成“先看摘要再决定是否精读”。这套管线的核心不在模型，而在去重、清洗、输出约束和推送策略。先把这些基础工程做扎实，后面接 OpenClaw、MCP 或任何 Agent 能力都会顺畅很多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/272d62d9458d6688.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/b166706664e77098.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/6054677b2b42b220.png)

