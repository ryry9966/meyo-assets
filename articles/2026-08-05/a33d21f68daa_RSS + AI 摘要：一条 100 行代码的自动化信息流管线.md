---
title: RSS + AI 摘要：一条 100 行代码的自动化信息流管线
feedId: 31653
source: 综合讨论
publishedAt: 2026-08-05
---

## 背景

信息获取的痛点不在于“没有信息”，而在于“噪音太多”。每天订阅了几十个技术博客、GitHub Release 和社区论坛 RSS，但真正打开阅读的不到 10%。90% 的时间浪费在“扫描标题 -> 点开 -> 发现没价值”的循环里。

我希望构建一条轻量管线：RSS 抓取 -> 大模型粗筛/摘要 -> 推送到 agent 工作流，让 OpenClaw 或自己的 agent 成为信息过滤层，输出可直接消费的精华列表。

## 做法

整个方案用 Python + feedparser + OpenAI 兼容接口实现，核心思路是“批量扫描后单次聚合总结”，而非逐条调用 LLM——逐条调用既贵又慢。

**第一版：粗暴摘要**

最直白的做法：抓 RSS 后把每条标题+链接+简介发给 LLM，让它逐个打分和摘要。

```python
import feedparser

def fetch_rss(url, limit=20):
    feed = feedparser.parse(url)
    entries = []
    for e in feed.entries[:limit]:
        entries.append({
            "title": e.get("title", ""),
            "link": e.get("link", ""),
            "summary": e.get("summary", "")[:200]
        })
    return entries
```

把每条条目拼装后请求模型。问题很明显：订阅 50 个源，每个源 20 条，就是 1000 次请求，费用和延迟都受不了。

**第二版：分治汇聚**

改进策略——每个源先单独聚合，得出该源的关键内容；最后把各源的汇总结果再次压缩，形成日报。

```python
def summarize_entries(source_name, entries, client):
    joined = "\n\n".join(
        f"Title: {e['title']}\nLink: {e['link']}\nSummary: {e['summary']}"
        for e in entries
    )
    prompt = f"""
    你是技术情报分析师。以下是来自 {source_name} 的最新条目。
    请筛选出其中最有价值的 3-5 条（技术含量高、非广告、有实际参考价值），
    用中文输出，每条格式：
    - 标题（链接）
      为什么值得看（一句话）
    """
    resp = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt + joined}]
    )
    return resp.choices[0].message.content
```

使用 gpt-4o-mini 节省成本，单次输入控制在 1500 token 以内。50 个源就是 50 次请求，压到可接受范围。

**第三版：接入 OpenClaw**

摘要生成后，直接通过 `mcp` 工具调用写入 Markdown 文件，或者回调到 agent。核心不再是“推送”，而是让 agent 在需要时拉取处理结果。这部分用 websocket 回调即可，不依赖 OpenClaw 的现有实现，自己维护一把更可控。

## 踩坑点

1. **RSS 的 summary 字段不可信**。有些源不给 summary，有些给的是一整页 HTML。用正则剥掉标签，截断 200 字以内，否则 prompt 成本飙升。

2. **重复标题轰炸**。同一篇文章会同时出现在 Planet 聚合源和原博客源里。需要对标题做简单去重——维护一个 hash 集合，标题的归一化字符串（小写去空格）去重。

3. **LLM 的幻觉和偏执**。模型会把两篇不同文章错认为同一篇，也会因为 prompt 里“技术含量高”而偏向 AI 相关主题，产生信息茧房。缓解办法：在 prompt 里明确“不要筛选热门话题，尽量保留覆盖面”，并保留原文标题在输出里供验证。

4. **时间处理**。不同源的时间格式五花八门，`dateutil.parser.parse` 处理不了全部异常。用 `try/except` 兜底，解析失败就跳过该条——不值得为个别条目中断整个流程。

## 可复用建议

- 把订阅源按主题分组（AI、DevOps、语言生态、安全），每个组单独跑一次摘要，避免混在一起。
- 输出格式固定为 YAML/Markdown 结构化文件，方便后续接入 OpenClaw 做进一步处理或存储到数据库。
- RSS 源会死。用 `feedparser` 抓取时设置超时（比如 10 秒），失败后标记并跳过，连续失败 3 次就把源移入待观察列表。
- 对用户而言，最实用的体验是“每天早上一份 5 分钟读完的技术日报”。不用做推送，一个静态文件就够——想读了自己拉。
- 预算控制：用 4o-mini 或本地 qwen2.5 7B 级别的模型跑摘要，日均成本低于几分钱。对延迟敏感的场景，流式输出可优化体验。

## 总结

这套管线的核心价值在于把“订阅-阅读-筛选”这个高注意力成本动作，拆解为“抓取-汇总-总结”的低成本自动化。大模型的价值不在于替代你阅读，而在于做第一层粗筛——这符合人的注意力分配原则。

最终产物可以很朴素：一个 Markdown 文件，或者一个简单网页。关键是稳定的定时任务和可控的成本预算。技术含量不高，但实用价值很高。建议从自己的信息流开始，逐步调整 prompt，找到适合自己的过滤粒度。

---

