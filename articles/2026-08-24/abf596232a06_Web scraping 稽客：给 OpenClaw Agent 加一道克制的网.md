---
title: Web scraping 稽客：给 OpenClaw Agent 加一道克制的网页采集层
feedId: 34493
source: 综合讨论
publishedAt: 2026-08-24
---

# Web scraping 稽客：给 OpenClaw Agent 加一道克制的网页采集层

## 背景

Agent 在 OpenClaw 里经常需要读取网页：查文档、看状态页、解析某个公开页面。常见的做法是把 `requests.get(url)` 包成一个 MCP 工具直接暴露给模型。结果通常会遇到几类问题：HTML 太长撑爆上下文、JS 渲染页面拿到空内容、触发目标站风控、误把登录态或内网地址带出去、请求不可审计。

所以问题不是“能不能抓”，而是“如何安全、可审计、可恢复地抓”。我把这个采集层叫“稽客”——一个只做读取、限域、限速、清洗内容的网页读取工具。

## 一、先定义边界

实现前先确定四件事：

1. **协议限制**：只允许 `http` 和 `https`，拒绝 `file://`、`gopher://` 等。
2. **域名白名单**：默认拒绝，显式配置允许的 host，支持子域名匹配。
3. **资源上限**：单页 HTML 拉取上限 2MB，提取后纯文本上限 50KB，不默认下载图片。
4. **权限与审计**：工具本身只读；记录调用者、目标 URL、最终 URL、状态码、耗时、缓存命中。

如果 OpenClaw 侧支持工具权限策略，应把采集工具声明为只读，并按操作域做限制。

## 二、实现一个 MCP 采集工具

核心依赖可以用 `httpx` + `selectolax` 或 `BeautifulSoup`。如果需要 JS 渲染，再单独接 Playwright，但不要默认启用。

```python
import httpx
from selectolax.parser import HTMLParser

class SafeScraper:
    def __init__(self, allowed_hosts: set[str]):
        self.allowed_hosts = allowed_hosts
        self.client = httpx.Client(
            timeout=10,
            follow_redirects=True,
            headers={
                "User-Agent": "OpenClaw-SafeScraper/1.0 (+local-agent)",
                "Accept": "text/html,application/xhtml+xml",
            },
        )

    def fetch(self, url: str) -> dict:
        self._check_url(url)
        resp = self.client.get(url)
        resp.raise_for_status()

        # 限制体积，避免异常页面打爆内存
        content = resp.content[:2_000_000]
        text = self._extract_text(content)

        return {
            "url": str(resp.url),
            "status_code": resp.status_code,
            "title": self._extract_title(content),
            "text": text[:50_000],
            "truncated": len(text) > 50_000,
        }
```

`_check_url` 要做两件事：一是 host 必须在白名单内；二是解析后的 IP 不能是私网、环回或保留地址，避免 SSRF。重定向后的最终地址也要再次校验，不能只校验初始 URL。

`_extract_text` 可以先去掉 `script`、`style`、`nav`、`footer`，再提取标题、段落和列表。返回给模型的应该是清洗后的 Markdown 或纯文本，而不是 raw HTML。

MCP 工具定义可以这样写：

```json
{
  "name": "scrape_url",
  "description": "Read and extract main text from a public web page. Only allowed domains. Returns cleaned Markdown, max 50k chars.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "url": {"type": "string", "format": "uri"},
      "mode": {"type": "string", "enum": ["auto", "html", "text"], "default": "auto"}
    },
    "required": ["url"]
  }
}
```

## 三、接入 OpenClaw 的实践步骤

1. 把 MCP server 作为本地进程或容器运行，白名单通过环境变量注入，例如 `ALLOWED_HOSTS=docs.example.com,status.example.com`。
2. 在 OpenClaw 的 MCP 客户端注册该 server，并开启只读策略。
3. 先只放一个测试域名，跑“查文档”和“查状态页”两条链路，观察模型调用是否稳定。
4. 增加缓存：以 URL + 提取模式为 key，TTL 建议 5–15 分钟；错误响应缓存短一些，避免反复打目标站。

## 四、踩坑点

- **直接抓 HTML 不等于可读内容**：很多文档站是 JS 渲染。可以先判断 `title` 是否为空，必要时再走 Playwright，并设置页面加载超时和资源拦截。
- **登录态泄露**：默认不携带 Cookie。认证站点需要单独受限工具，不要和通用 scraper 混用。
- **SSRF 与内网探测**：攻击者可能通过重定向、userinfo、端口等方式探测内网。必须解析最终地址并再次校验 host/IP。
- **频率与并发**：模型可能一次调用多个 URL。加全局并发限制和 per-host 限速，如 2 并发、1 req/s。对 429 返回 `retry-after` 给模型，而不是直接失败。
- **大页面撑爆上下文**：设置 `max_text_chars` 后模型可能只看到一部分。返回里明确标注 `truncated`，并提供 `offset` 参数或分块提示。
- **URL 去重**：去掉 `utm_*`、`fbclid` 等追踪参数后再做缓存 key，否则同一内容会被抓多次。

## 五、可复用建议

- 用一个 `ScrapePolicy` 配置对象统一管理白名单、体积限制、超时和重试，不要散落在工具函数里。
- 错误结构化：返回 `{"ok": false, "error": "domain_not_allowed", "hint": "..."}`，不要让异常栈直通模型。
- 提供三个工具：`scrape_url`、`scrape_status`、`scrape_cache_clear`，方便调试和刷新。
- 每次抓取记录一行审计日志：时间、调用者、目标 URL、最终 URL、状态码、响应大小、耗时、缓存命中。
- 工具描述里写清限制，例如“Only public pages; returns cleaned Markdown, max 50k chars”，减少模型误用。
- 先做读取，不做写入。任何表单提交、点击、登录都放到单独工具，并加二次确认。

## 总结

“稽客”不是万能爬虫，而是给 Agent 一个克制的网页读取边界。核心是把“访问权限”和“内容消费”解耦：用白名单、域名/IP 校验、体积限制控制访问；用正文提取、缓存、截断控制上下文。对 OpenClaw 用户来说，一个稳定的 MCP scraper 应该让模型拿到的不是 HTML，而是可读的、有限的信息，并且每次访问都可追溯。这样才能让 Agent 获得网页采集能力，又不把自动化变成失控爬虫。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/b4206ea69d026b79.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/73af8d9a7ea6faeb.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/1c0d0efd9a0fb9ef.png)

