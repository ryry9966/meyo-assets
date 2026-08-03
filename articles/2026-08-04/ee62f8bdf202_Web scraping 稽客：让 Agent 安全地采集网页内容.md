---
title: Web scraping 稽客：让 Agent 安全地采集网页内容
feedId: 31519
source: 综合讨论
publishedAt: 2026-08-04
---

## 当 Agent 开始乱抓网页

OpenClaw 社区里最常见的自动化动作，不是调 API，而是让 Agent 去读一个网页。OpenAI 的 MCP 生态起来之后，很多人给 Agent 塞了一堆 fetch、browser、playwright 工具，结果发现 Agent 是会把整个页面塞进 context 的，甚至会把登录页、反爬页、404 页当有效内容处理。

我管这个问题叫「抓取失控」。Agent 有自主性，但它不懂 Web 江湖的潜规则：robots.txt、限流、版权、内容协商。如果你的 Agent 每天要采集几十个页面，这些「潜规则」就不是玄学，而是稳定性问题——IP 被封、内容错乱、context 爆炸。

这篇文章聊的是怎么当「稽客」：不追求抓得最多，追求抓得合规、抓得稳、抓得可复现。

## 核心问题：Agent 不是可靠的采集器

有几个实际踩过的坑：

1. **反爬识别**。无头浏览器默认 UA 带上 `HeadlessChrome`，很多站一眼识破。而普通 curl 带上浏览器 UA，反而能拿到静态页面。
2. **内容协商失效**。Agent 直接抓 HTML，不懂 `Accept: text/markdown` 这类 header，抓到一堆导航、广告、注释，把真正的内容稀释了。
3. **abort 和重试失控**。Agent 发现抓超时了，自己重试三次，等于帮对方刷了三次量，IP 更容易被 ban。
4. **robots.txt 被忽略**。我不讨论法律，我只说现实：有些站点的访问统计就是通过日志做的，你不在乎 robots，对方就懒得给你好脸。

这些问题里，第 1、2 条是技术问题，第 3、4 条是工程治理问题。稽客的核心，就是把这四件事做成一个固定流程。

## 做法：一个可控的采集技能（Skill）

我建议用 OpenClaw 的 Skill 机制，写一个「稽核采集器」，而不是让 Agent 直接调 curl 或 MCP fetch 工具。核心流程如下：

### 1. 预检（Preflight）

Agent 在真正抓取之前，先请求 `robots.txt`，检查目标路径是否允许。同时检查 `sitemap.xml`，如果目标 URL 不在 sitemap 里，但 robots 允许，也放行。

这一步很便宜，但对合规和防封都有实际意义。再设置一个全局抓取频率上限，比如每分钟不超过 5 个页面。

### 2. 剥离 HTML → 正文提取

不要直接把 HTML 塞给 Agent。在 Shell 技能里，先用 `curl -s -L --compressed` 拿 HTML，再交给一个专门的正文提取器处理。

推荐 Readability（Mozilla 的开源库）的 Python 实现，或者用 `readabilipy`；不想装依赖，也可以用 `python3 -c "import readability"` 的最小封装。我自己的做法：

```python
import requests
from readability import Document

resp = requests.get(
    url,
    headers={
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36",
        "Accept-Language": "zh-CN,zh;q=0.9",
    },
    timeout=10,
)
doc = Document(resp.text)
title = doc.shorten_title()
content = doc.summary(html_partial=True)
```

然后把 `content` 里面的 HTML 标签去掉，转成纯文本，限制长度（比如前 3000 字），同时保留原文链接和抓取时间。

### 3. 落盘为「带来源的证据」

Agent 拿到的不该是一段裸文本，而是一个结构化条目：

```markdown
来源：https://example.com/post/123
抓取时间：2025-06-20 10:22:01
标题：xxx
- 正文前 3000 字 -
```

这一步很关键：Agent 后续做总结或引用时，有据可查，不会把不同来源的内容混在一起说。

### 4. 缓存与去重

把抓取结果按 URL hash 存到本地目录（比如 `~/.openclaw/cache/`，48 小时有效）。Agent 再次请求同一个 URL 时，直接读缓存，不重新发起网络请求。

这一点对降本和防封效果最直接。

## 踩过的坑

讲几个真实的：

1. **`--compressed` 忘了加**。有的站返回 gzip，你看到的是乱码。Mac 自带的 curl 默认不自动解压，一定要加。
2. **不要用 Playwright 做默认方案**。很多站点对无头浏览器的行为检测更严格，权重高的响应 CDN 会在 TLS 指纹层面拦你。纯静态页面用 curl 更稳。Playwright 只留作最后的视觉兜底。
3. **Readability 的摘要不适合代码或表格页**。文章页效果很好，但是文档站、GitHub README 会被砍得残缺。需要根据域名做一个规则映射：文档类站点直接提取 `<article>` 或 `<main>` 内的全部文本。
4. **不要让 Agent 自己决定重试逻辑**。Agent 遇到超时时习惯性重试。在我的 Skill 里，把重试次数写死为 0，超时就放弃并记录 URL，放到「下次补采」队列里。

## 可复用的建议

- **把抓取白名单做成机制**。你的 Agent 只允许采集你主动提供的 URL，或者那些在预检阶段通过校验的 URL。不让 Agent「顺藤摸瓜」随便跳转。
- **对抓取结果做文本指纹**。以内容前 200 字做 hash，存到 SQLite 里，能够快速发现同文镜像、站内转载，方便合并。
- **定时任务配合队列**。如果采集量有持续需求，建议用简单队列（Shell 脚本足够），不依赖 Agent 的一次性对话 session。OpenClaw 的定时调度可以配合起来，每天凌晨跑一次增量采集。
- **所有请求记录日志**。包括状态码、耗时、字节数、缓存命中。过两周回头看，能发现很多规律。

## 总结

稽客不是「不要采集」，是把采集从 Agent 的临时行为变成一套可审计、有边界、有缓存的工程流程。

真正落地时，你会发现 Agent 的「聪明」是在流程之外的。OpenClaw 的价值就在于它允许你把这种流程固化成 Skill，让 Agent 在框架里发挥，而不是裸奔抓网页。这比给它一个 MCP fetch 工具，然后寄希望于它自觉，要可靠得多。

---

