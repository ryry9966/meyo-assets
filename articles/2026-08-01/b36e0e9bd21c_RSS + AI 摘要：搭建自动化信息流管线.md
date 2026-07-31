---
title: RSS + AI 摘要：搭建自动化信息流管线
feedId: 31148
source: 综合讨论
publishedAt: 2026-08-01
---

## 背景：信息输入正在吞噬我们的注意力

每天几十个 RSS 源、数百篇文章，加上新闻信、社交媒体，人工判断「是否值得读」的成本已经到了不堪重负的地步。RSS 本身是一种干净的去中心化协议，但缺少自动筛选与压缩的能力。AI 摘要恰好填补了最后一公里——将源头内容浓缩成 150 字，让你 30 秒内做出阅读决策。

这个帖子里，我会给出一条可落地的自动化管线：从 RSS 采集到全文提取，再经过本地/远程的大模型摘要，最后推送到你日常使用的工具里。所有组件尽量选可自托管、低成本的方案。如果你已经在玩 OpenClaw/MCP/插件化 Agent，这里的工程思路和踩坑清单应该可以复用。

## 问题拆解

一条稳健的信息流管线至少要处理四件事：

1. **可靠的内容获取**：RSS 标题+摘要经常是截断的，需要抓回全文才能做有效摘要。
2. **可控的 AI 调用**：调用次数和 token 消耗要能压下来，否则每月账单比云服务器还贵。
3. **去重与噪音过滤**：同一事件被多个源头报道，重复摘要没有价值。
4. **送达与保存**：推送到 IM、笔记或稍后读工具，避免又回到“推送太多不想看”的困境。

下面我会给出一个基于 Miniflux、Python 和 Ollama 的架构，你可以按需替换组件。

## 方案鸟瞰

- **聚合与触发**：Miniflux（自部署 RSS 阅读器）负责订阅、更新、去重，并通过 Webhook 在发现新文章时触发外部服务。
- **全文提取**：使用 `readabilipy` 或 `trafilatura` 从原始 URL 中抽取正文。
- **AI 摘要**：优先用本地 Ollama 跑 7B 模型（Mistral/Llama3），如果速度不够再回落 OpenAI API。
- **输出端**：通过 Telegram Bot 或飞书 Webhook 发送摘要，同时可选写入 Notion 数据库留档。
- **调度与可靠性**：一个小型 Python 服务，接收 Miniflux webhook，执行全文抓取+摘要，带重试和去重逻辑。

这样做的好处是：“触发”比定时轮询更实时，且 Miniflux 已经做了很多脏活（解析 feed、更新频率控制、条目去重）。

## 实现步骤

### 1. 部署 Miniflux 并接入 Webhook

推荐使用 Docker Compose 一键拉起：
```yaml
services:
  miniflux:
    image: miniflux/miniflux:latest
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=postgres://miniflux:secret@db/miniflux?sslmode=disable
      - RUN_MIGRATIONS=1
      - CREATE_ADMIN=1
      - ADMIN_USERNAME=admin
      - ADMIN_PASSWORD=changeme
    depends_on:
      - db
  db:
    image: postgres:15
    environment:
      - POSTGRES_USER=miniflux
      - POSTGRES_PASSWORD=secret
    volumes:
      - miniflux-db:/var/lib/postgresql/data
```
登录后导入 OPML，然后在 Settings > Integrations 中配置一个 Webhook URL。Miniflux 会在新条目入库时 POST 一个 JSON，包含标题、URL、内容（可能不完整）等。

### 2. 搭建中间处理服务

写一个简单的 Python 服务（FastAPI 即可），暴露 `/new-entry` 端点：

```python
import trafilatura
import requests
from ollama import Client
import hashlib
import json

OLLAMA_CLIENT = Client(host='http://localhost:11434')
SEEN = set()  # 生产环境改用 Redis

def extract_full_text(url):
    downloaded = trafilatura.fetch_url(url)
    if downloaded:
        return trafilatura.extract(downloaded, include_links=False, include_images=False)
    return None

def summarize(text):
    response = OLLAMA_CLIENT.chat(
        model='mistral:7b',
        messages=[{'role': 'user', 'content': f"用不超过150字总结以下文章要点，保留关键数据和结论：\n\n{text[:3000]}"}]
    )
    return response['message']['content']

@app.post('/new-entry')
def handle_entry(payload: dict):
    url = payload['entry']['url']
    title = payload['entry']['title']
    if url in SEEN:
        return 'ok'
    SEEN.add(url)
    full_text = extract_full_text(url)
    if not full_text:
        # 退回使用 Miniflux 提供的摘要
        full_text = payload['entry'].get('content', '')
    summary = summarize(full_text)
    send_to_telegram(f"{title}\n\n{summary}\n🔗 {url}")
    # 可选写入 Notion
    return 'ok'
```

### 3. 配置输出渠道

以 Telegram 为例，创建 Bot 后得到 token，发送消息只需：
```python
def send_to_telegram(text):
    requests.post(f'https://api.telegram.org/bot{TOKEN}/sendMessage', json={
        'chat_id': CHAT_ID, 'text': text, 'disable_web_page_preview': False
    })
```
你也可以替换成飞书、钉钉，或者通过 MCP Server 写入 Obsidian——只要抽象成一个函数即可。

## 踩坑与工程化要点

- **全文提取失败率出乎意料**：不少网站有反爬，或者使用 JS 渲染。`trafilatura` 对静态内容效果最好，但遇到动态页面需要 fallback 到浏览器渲染（如 Playwright）或直接用 Miniflux 的摘要。约定一个降级逻辑，别让管线死在某个坏链上。
- **AI 费用/性能的双重陷阱**：不停调用 GPT-4 每个文章花掉几毛钱，RSS 量一上去很快破百。更合理的是用 7B-13B 级别的本地模型，或者只对**英文**源用 OpenAI，对中文源用 Qwen-7B 或 DeepSeek。还要注意 prompt 里限制输入长度，否则大文本会撑爆 context。
- **去重不能只靠 URL**：同样的新闻可能出现在不同域名，你需要的不仅是 URL 去重，还要做标题或内容相似度判断。可以用局部敏感哈希（如 MinHash + LSH）但轻量级方案用 `difflib` 计算标题相似度超过 0.8 就跳过。
- **Webhook 可靠性**：Miniflux 的 webhook 没有重试机制。如果你自己的服务宕机，会丢消息。建议在中间服务前面加一层轻量队列（Redis + RQ 或甚至 `systemd` 的 socket activation），以保证消息被持久化处理。

## 可复用建议

1. **MCP 化接口**：把上述处理服务封装为 MCP server 的 tool，这样 Claude Desktop 或其他 Agent 可以直接查询“今天未读的 AI 摘要”。你可以在 OpenClaw 里编排一个每日简报 Agent，它会调用 MCP tool 获取摘要并生成日报。
2. **按源设置策略**：不是所有源都需要摘要。高频短资讯（如 Hacker News）只需要标题+链接；深度长文才值得走 AI。在 Miniflux 里可以按 Category 打标签，webhook 中区分处理。
3. **监控与兜底**：加一个简单的健康检查 cron，每小时统计一次处理成功率和延迟，超过阈值自动降低处理频率或切到 OpenAI 兜底模型。
4. **标注信心指数**：在摘要末尾加一个小标记，比如“模型信心：高/中/低”，由 AI 自评，帮助你快速跳过质量差的摘要。

## 总结

一条成熟的 RSS + AI 摘要管线，本质是把“阅读决定权”提前到摘要阶段，而不是在原文上浪费整块时间。选型时可以自托管 Miniflux 解决源的稳定性，用 trafilatura 补全文提取，AI 负担由本地模型扛下，最后通过 webhook 或 MCP 工具融入你现有的信息食粮。

整个架构的核心不是惊天算法，而是对故障和成本有节制的处理：去重、降级、队列、固定 prompt。做到这四点，你就拥有了一条真正降噪、而非增噪的信息管线。

---

