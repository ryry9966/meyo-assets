---
title: RSS + AI 摘要：自动化信息管线从折腾到稳定
feedId: 31928
source: 综合讨论
publishedAt: 2026-08-07
---

## 为什么要自己搭 RSS + AI 摘要管线

信息过载的解法很多，但 RSS 始终是最可被机器消费的源头之一。你把关注的博客、新闻、期刊、社区动态全部丢进一个阅读器，理论上就能告别算法推荐。然而真正跑起来的人都知道：每天几百条未读，看完标题就累了，更别提精读。

AI 摘要似乎能破局——让模型帮你做第一轮过滤，只把值得看的推送到眼前。但现成的 SaaS 方案要么贵、要不自定义弱，开源工具又需要缝合。本文记录一条低成本、可长期运行的管线，适合已经玩 OpenClaw、Agent、MCP 这类自动化组件的你。

## 问题拆解

我们要解决的是一串工程问题，而不是“调个 API 完事”：

- 如何可靠地从多个 RSS/Atom 源抓取内容，处理好编码、重定向；
- 如何把文章正文（或部分 HTML）结构化，准备给 LLM；
- 怎样让 LLM 输出可控的摘要，不幻觉、不浪费 token；
- 怎样推送摘要到微信/Telegram/飞书，还能防止重复轰炸；
- 任务调度、状态持久化、低维护成本。

最终目标：每天定时跑一次，把聚合后的“今日要闻”推送到你的聊天工具，附带原文链接，方便深读。

## 做法与步骤

### 1. 整体架构

```
OPML/配置文件 -> 抓取模块 -> 去重 & 结构化 -> LLM 摘要 -> 推送渠道
```

调度可以用 cron、GitHub Actions，也可以在 OpenClaw 中创建 Agent，用定时触发器驱动。这里给一条轻量级 Python 实现路径，所有组件都可以替换成你熟悉的工具。

### 2. 数据抓取

用 `feedparser` + `requests`，配合 `chardet` 处理中文源常出现的 GB2312 乱码：

```python
import feedparser, requests, chardet

def fetch_feed(url):
    resp = requests.get(url, timeout=15)
    if resp.encoding is None:
        guessed = chardet.detect(resp.content)
        resp.encoding = guessed['encoding'] or 'utf-8'
    return feedparser.parse(resp.text)
```

维护一个 OPML 文件，或直接存成 JSON 列表。每次抓取时记录文章的 GUID 或（link, title）组合作为去重键，存储在本地 SQLite / txt 文件里。已有 GUID 的条目直接跳过，避免重复推送。

踩坑：部分 RSS 源的链接会变（加追踪参数），所以用 `link.split('?')[0]` 做归一化。GUID 有时竟然是空的，那就用 `link + title` 的 hash。

### 3. LLM 摘要生成

设计一个克制但信息密度高的 prompt，固定输出字段：

> 你是一个信息筛选助手。根据以下文章内容，提取：
> 1. 一句话主旨
> 2. 三个关键点（每条不超过40字）
> 3. 标签（1-3个）
> 
> 只返回 JSON，不要多余解释。

将 `feedparser` 得到的 `summary` 或 `content[0].value` 转为纯文本，截断前 4000 字符（够用）。调用 OpenAI 兼容的 API（可以用自部署的模型，只要同接口）。控制 `max_tokens` 在 300 左右，避免浪费。

如果你在用 OpenClaw 的 Agent 能力，可以直接把这个 prompt 封装成一个 Function Call Tool。后续通过 OpenClaw 内置的 LLM 连接器调用，连 API key 都不用自己管。

### 4. 推送

推荐 Telegram Bot，因为 API 极简，且支持 Markdown。直接发送：

```
**{{主旨}}**
标签: {{标签}}
1. {{关键点1}}
2. {{关键点2}}
3. {{关键点3}}
[阅读原文]({{link}})
```

想推微信可以借助企业微信机器人 Webhook，或者通过 OpenClaw 已有的推送插件。别逐条推送——收集当天所有摘要后合并成一条长消息，减少骚扰。

### 5. 整合进 OpenClaw

最近 OpenClaw 的插件体系已经能直接挂载 Python 脚本。你可以：

- 创建一个 Agent，定义 `fetch_and_summarize` 技能。
- 技能内调用上述 Python 逻辑，或者用 requests 调一个小型 HTTP 服务。
- 用 OpenClaw 的定时任务 trigger，每天早八点触发一次。
- 结果通过 OpenClaw 的 Message Sender（Telegram / 飞书等）直接推送。

更进一步，把“已读池”管理封装成一个 MCP Server，供其他 Agent 调用——这样你的信息流就成了基础设施，不只是个人工具。

## 踩坑记录

- **编码地狱**：中文 RSS 大概率遇上非 UTF-8，不检测就全是乱码。`chardet` 是救星，但也要给 `feedparser` 直接传解码后的文本。
- **HTML 转纯文本**：用 `html2text` 库，注意处理 `<pre>` 代码块，否则代码被挤成一句。
- **模型幻觉**：有时 LLM 会捏造原文不存在的内容。加了“只基于原文内容”的约束后明显好转，但无法根治。建议保留原文链接方便校对。
- **推送频率**：初期每抓一条就推一次，结果被 Telegram 限速。改成每日汇总后安静很多。
- **GUID 去重持久化**：第一次跑时忘了存 GUID，导致每次推送都重复。务必在每次成功抓取后立刻写入去重记录。
- **时区**：GitHub Actions 默认 UTC，推送时间容易错位。设置 `TZ=Asia/Shanghai` 并确认 cron 表达式。

## 可复用建议

1. **将 feed 列表外置**为 YAML 文件，方便增删源而不改代码。
2. **模型切换**靠环境变量 `OPENAI_BASE_URL` 和 `MODEL_NAME`，想换本地模型只需改两行。
3. 把整个流程做成一个 OpenClaw **可共享插件**，只需提供 OPML 和 Webhook URL 即可用。
4. 如果团队使用，可以为每个人维护一个已读池，避免同事收到相同文章时觉得扰民。
5. 定期清理过期 GUID（超过 7 天的记录可以删），保持去重库轻量。

## 总结

RSS + AI 摘要这个组合，表面上看只是个“懒得看全文”的工具，实际落地时会踩一堆工程坑。但一旦管线稳定运行，你的信息摄入模式会从被动刷屏变成主动抽样，阅读效率的提升是实在的。对于已经在用 OpenClaw 做自动化的用户来说，把这条管线集成到自己的 Agent 工厂里，成本很低，效果却立竿见影。

如果你想把这套东西做成 MCP 服务分享出来，欢迎发到 OpenClaw-CN 社区一起打磨。

---

