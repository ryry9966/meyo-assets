---
title: RSS + AI 摘要：搭建自动化信息流管线
feedId: 31780
source: 综合讨论
publishedAt: 2026-08-06
---

## 背景：信息过载与结构化机会

折腾自动化管线的人，迟早会撞上信息过载这个老问题。源越来越多，订阅列表越来越厚，真正能读完的内容并没有同步增长。把 RSS 重新捡起来，配上一个轻量级的 AI 摘要步骤，是一条实践门槛不高、收益却很实在的路径。

这篇帖子记录了我用 RSS 汇聚 + LLM 摘要搭建信息流管线的过程，面向已经在玩 OpenClaw、MCP 或本地 Agent 的同学。整套方案不依赖任何第三方黑盒服务，所有组件都可以自己跑，也可以按需替换为社区现有插件。

## 要解决的问题

我的诉求很具体：每天需要关注十几个技术站点、几个论文源和若干博客的更新，但每天能花在信息筛选上的时间不超过 30 分钟。于是管线需要完成三件事：

1. **聚合**：把分散的 RSS/Atom 源汇到一起，支持增量抓取和去重。
2. **摘要**：对文章正文做结构化总结（标题、核心观点、关键数据），长度控制在 200 字以内，方便快速判断是否需要精读。
3. **分发**：将处理结果推送到我日常使用的渠道（Telegram 频道、邮件日报或者一个独立的摘要 feed）。

额外约束：管线必须在自己可控的计算资源上跑，所有密钥本地管理，不做数据外泄。

## 选型与架构

核心组件选型如下：

- **RSS 抓取**：`feedparser` + `requests`，足够处理绝大多数标准源。需要处理非标源时，配合 `rsshub` 容器本地部署作为补充。
- **正文提取**：多数 RSS item 的 `summary` 字段要么只有几句话，要么塞满 HTML。用 `trafilatura` 或 `readability-lxml` 从原文链接抓取全文，比直接清洗 RSS 摘要更可靠。
- **LLM 摘要**：本地跑 `ollama` + `qwen2.5:7b` 处理常规文章，对长文或论文改用 OpenAI API（gpt-4o-mini），兼顾速度和精度。
- **编排**：用 Python 脚本串联整个流程，通过 cron 定时执行。没有引入 n8n 或 Airflow，因为管线只有三步，直接写脚本反而是最可控的方式。
- **去重与缓存**：SQLite 记录已处理文章的 `link` 字段哈希，避免重复摘要和推送。本地文章缓存 30 天，方便回溯。
- **分发**：Telegram Bot 推送摘要卡片，同时生成一份静态 RSS 摘要文件，由本地 nginx 暴露为 RSS 源（可在 RSS 阅读器里订阅自己的加工结果）。

流程图简示：

```
RSS源(多个) → feedparser 抓取 → 去重 → trafilatura 提取正文
                                       ↓
                                LLM 摘要(本地/API)
                                       ↓
                                  SQLite 存储
                                       ↓
                        Telegram 推送 + 生成摘要 RSS
```

## 关键步骤与代码骨架

### 1. 抓取与去重

```python
import feedparser
import hashlib

def fetch_new_items(url, seen_hashes):
    feed = feedparser.parse(url)
    new_items = []
    for entry in feed.entries:
        link_hash = hashlib.sha256(entry.link.encode()).hexdigest()
        if link_hash in seen_hashes:
            continue
        new_items.append(entry)
        seen_hashes.add(link_hash)
    return new_items
```

`seen_hashes` 从 SQLite 加载，处理后存回数据库。

### 2. 正文提取

很多 RSS 源只在 `summary` 里放了前几行，直接丢给 LLM 效果很差。用 `trafilatura` 从源 URL 抓取：

```python
import trafilatura

def extract_full_text(url, fallback_text=''):
    downloaded = trafilatura.fetch_url(url)
    if downloaded:
        text = trafilatura.extract(downloaded, include_comments=False,
                                   output_format='markdown')
        if text:
            return text
    return fallback_text
```

超时和网络异常要兜底，降级使用 RSS 的 `summary`。

### 3. LLM 摘要

根据文章长度路由到本地模型或 API：

```python
def summarize(text, max_tokens=200):
    if len(text) < 2000:
        model = 'local'  # ollama
    else:
        model = 'api'    # gpt-4o-mini
    prompt = (
        "Summarize the following article in Chinese, within 200 characters. "
        "Focus on main topic, key findings, and actionable insights.\n\n"
        f"{text}"
    )
    if model == 'local':
        import ollama
        resp = ollama.chat(model='qwen2.5:7b', messages=[{'role':'user','content':prompt}])
        return resp['message']['content']
    else:
        import openai
        client = openai.OpenAI()
        resp = client.chat.completions.create(
            model='gpt-4o-mini',
            messages=[{'role':'user','content':prompt}],
            max_tokens=max_tokens
        )
        return resp.choices[0].message.content
```

注意控制 prompt 长度，避免超出模型上下文。

### 4. 分发到 Telegram 并生成摘要 RSS

Telegram 推送直接用 Bot API。生成摘要 RSS 可以用 `feedgen` 库：

```python
from feedgen.feed import FeedGenerator

fg = FeedGenerator()
fg.title('我的 AI 摘要 RSS')
fg.link(href='https://myhost/summary.xml')
fg.description('由 RSS + LLM 自动生成的技术摘要')

# 从 SQLite 读取最近 20 条摘要写入 feed
for item in recent_items:
    fe = fg.add_entry()
    fe.title(item['title'])
    fe.link(href=item['link'])
    fe.description(item['summary'])
fg.rss_file('summary.xml')
```

## 踩坑记录

1. **RSS 格式混乱**：部分站点 `published` 字段缺失导致排序出错，需要回退到 `updated` 或抓取时间。媒体源（播客 RSS）的 `enclosure` 属性要提前过滤，否则 `trafilatura` 会误抓音频文件。
2. **正文提取失败率**：大约 15% 的源因为反爬、JS 渲染页面导致 `trafilatura` 拿不到正文。这种场景无法在脚本内解决，只能标记“摘要不可用”，或者人工添加备用摘要规则。不要在这个环节追求 100% 自动。
3. **LLM 摘要幻觉**：本地 7b 模型偶尔会把“作者推测”当成“实验结论”。必须配置严格的 prompt 约束，比如增加“如果原文未提供数据，不要编造”的指令，并在摘要末尾标注置信度标记。
4. **成本控制**：全用 API 处理，单日 50 篇文章约 $0.02，但如果忘了加文章缓存会导致重复调用，费用指数增长。SQLite 缓存是最廉价的防火墙。
5. **摘要质量参差**：技术文章摘要还行，但对评论区、免责声明之类的内容会浪费 token，需要用 `trafilatura` 的 `include_comments=False` 做第一道清洗。

## 可复用建议

- **模块化拆解**：抓取、提取、摘要、分发四个模块独立，可以分别替换。比如想把 LLM 换成本地部署的 MCP 服务，只需要改 `summarize` 模块的调用方式。
- **配置驱动**：用一个 YAML 文件管理所有订阅源、LLM 参数、推送渠道。加新源只是多一行 URL。
- **日志与监控**：每条管线执行留记录，摘要失败率一旦超过 20% 自动发通知。日志用 structlog 输出 JSON，方便以后接入可视化。
- **和 OpenClaw 社区工具的结合点**：如果已经搭了 MCP 服务器，可以把 RSS 抓取注册成 MCP 资源，LLM 摘要用 Agent 工具调用完成，整个管线就是一个可编排的 Agent 工作流。我目前没这么做是觉得脚本更少依赖，但可扩展性上 MCP 方案更优。
- **开源零件推荐**：`feedparser`, `trafilatura`, `feedgen`, `ollama`, SQLite。不做重复造轮。

## 总结

用 RSS + AI 摘要搭建的信息流管线，解决的核心问题是“用少量时间获得高密度信息”。它不算新鲜玩法，但在自己手里跑起来后，对信息的掌控感完全不同——你知道哪些内容经过了加工，哪些直接跳过，成本和延迟都透明。

这个管线的局限也很明确：正文提取无法 100% 自动化，摘要质量严重依赖 prompt 工程，推送渠道的排版适配需要额外维护。但它是一个典型的把 LLM 作为信息处理中间件的案例，后续可以很自然地生长出更多自动化动作，比如根据摘要自动归档到知识库、触发二次深度阅读，或者与 OpenClaw 的 Agent 联动做多步分析。

如果你也在搭类似的管线，建议从一小批高质量源开始，先跑通端到端流程，再逐步扩大规模。自动化信息处理的难点永远不在工具，而在定义“什么值得被处理”。

---

