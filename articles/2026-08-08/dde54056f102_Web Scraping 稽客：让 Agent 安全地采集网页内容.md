---
title: Web Scraping 稽客：让 Agent 安全地采集网页内容
feedId: 32070
source: 综合讨论
publishedAt: 2026-08-08
---

## 背景

在 OpenClaw 这类 Agent 框架中，让 Agent 直接访问网页并提取内容的需求越来越普遍。无论是需要获取实时数据做决策，还是帮用户总结某个页面，Agent 都需要一个“读网页”的能力。最简单的实现往往是直接发 HTTP 请求拿 HTML，再用 BeautifulSoup 或 Cheerio 提取文本，一股脑塞进上下文。

这个方案在小规模试验时足够快，但一旦放到生产场景，问题就会集中爆发：部分网站会拦截非浏览器请求、重定向到登录页或者返回空的 SP C壳；大体积页面会让上下文窗口迅速爆掉；更危险的是，Agent 可能跟着相对路径去请求不该访问的内网地址，或被注入恶意的 JavaScript 跳转利用。

换句话说，如果把“读网页”简单地等同于“下载 HTML 后喂给 LLM”，就等于把整个互联网的不确定性直接暴露给了 Agent。我们需要一个更安全的中间层，也就是本文要讨论的 **Web Scraping 稽客（Guard）**。

## 问题拆解

稽客的核心责任是：**把不可控的网页内容转换成 Agent 可安全消费的结构化信息**。具体要解决三层问题：

1. **环境隔离**：请求必须从受控的沙箱环境发起，避免暴露内网或宿主机信息。
2. **内容清洗**：要去除脚本、样式、跟踪像素、外部资源引用，合并拆分的段落，控制最终文本体积。
3. **权限收敛**：Scraper 只能访问显式许可的域名或 URL 模式，且禁止跟随任意重定向。

如果稽客作为 MCP 工具或插件嵌入 OpenClaw，Agent 调用的就不再是原始的“fetch_url”，而是一个安全加固后的接口。

## 实践方案

下面以 Python 生态为例，搭建一个可跑通的稽客服务，对外暴露两个核心能力：

- `scrape_text(url)` → 返回清洗后的纯文本，长度限制在 60K token 以内
- `scrape_markdown(url)` → 当页面结构良好时，尝试转换为 Markdown

### 1. 沙箱化请求

使用 `playwright` 或无头浏览器发起请求，而不是 `requests`。原因有三：能执行 JavaScript 拿到 SSR/SPA 的实际内容；可以模拟真实浏览器指纹降低拦截率；天然支持网络隔离。

```python
from playwright.async_api import async_playwright

async def fetch_rendered_html(url: str, timeout: int = 15000) -> str:
    async with async_playwright() as p:
        browser = await p.chromium.launch(
            headless=True,
            args=["--no-sandbox", "--disable-setuid-sandbox"]
        )
        context = await browser.new_context(
            user_agent="OpenClaw-Scraper/1.0",
            bypass_csp=True,
            ignore_https_errors=False
        )
        page = await context.new_page()
        try:
            await page.goto(url, timeout=timeout, wait_until="domcontentloaded")
            # 等待关键内容出现，避免拿到空白骨架屏
            await page.wait_for_selector("body", timeout=5000)
            html = await page.content()
            return html
        finally:
            await context.close()
            await browser.close()
```

**关键决策**：必须设置合理的 `timeout`、显式等待 `body` 元素，并关闭多余权限（如 `geolocation`、`notifications`），否则容易卡死或弹权限框。

### 2. 清洗与长度控制

拿到 HTML 后，不能直接交给 `BeautifulSoup.get_text()`，因为会把 `nav`、`footer`、`script` 里的文本也全拉出来，形成大量噪音。需要：

- 移除 `script`、`style`、`noscript`、`iframe`、`svg`
- 移除 `header`、`footer`、`nav` 等辅助区域（可通过标签或 ARIA role 识别）
- 对 `pre`/`code` 保留原格式，其余合并空白字符
- 提取主内容区：优先用 `article`、`main`、`[role="main"]`，若无则回到 `body`
- 截断：对清洗后文本做 token 估算（粗略按字符数 / 4），超过 60K token 则保留头尾，中间注明省略

```python
from bs4 import BeautifulSoup
import re

def clean_html(html: str, max_tokens: int = 60_000) -> str:
    soup = BeautifulSoup(html, "lxml")
    # 移除噪声标签
    for tag in soup(["script", "style", "nav", "footer", "iframe", "noscript", "svg"]):
        tag.decompose()
    # 定位主内容
    main = soup.find("main") or soup.find(role="main") or soup.find("article") or soup.body
    if not main:
        return ""
    text = main.get_text(separator="\n")
    # 压缩空行
    text = re.sub(r"\n\s*\n", "\n\n", text)
    text = text.strip()
    # 粗略 token 限制
    if len(text) // 4 > max_tokens:
        half = max_tokens * 2  # 约一半
        text = text[:half] + "\n\n...[content truncated]...\n\n" + text[-half:]
    return text
```

### 3. 权限与域名白名单

Scraper 绝不能做通用代理。每次调用时，必须校验目标 URL 是否属于允许的范围。可以在稽客服务启动时加载一个 `allowed_domains.yaml` 配置文件，格式：

```yaml
allowed:
  - domain: "docs.openclaw.ai"
    paths: ["/guide/", "/reference/"]
  - domain: "github.com"
    paths: ["/openclaw/"]
```

匹配逻辑可以用 Python 的 `urlparse` 提取 `hostname` 和 `path`，逐一比对。对于不在白名单内的请求，直接返回固定错误，不建立实际连接。

**禁止跟随重定向**：Playwright 默认会跟随，需设置 `context.route` 拦截 3xx 响应，或在 `page.goto` 前禁用自动跳转。一旦发现目标 URL 发生重定向到非白名单域，立刻终止并返回错误。

### 4. 作为 MCP 工具暴露

将上述能力封装成 MCP 兼容的 JSON-RPC 接口，这样 OpenClaw 的 Agent 就能以 `tool_call` 方式调用。核心方法：

- `scrape`：接收 url 和 format（text/markdown），返回结构化的 `{title, text, truncated, error?}`
- `health`：简单心跳

注意工具描述（schema description）里要明确告知模型：这个工具只能读公开 Web 页面，不能用于登录、提交表单、下载文件。这层提示词约束可以防止 Agent 滥用。

## 踩坑点

1. **浏览器实例泄露**：无头浏览器若没有正确关闭，容器内存会持续上涨。务必在 `finally` 块关闭 context 和 browser，并设置进程级别的超时兜底（如 `subprocess` 的 `timeout`）。
2. **页面永远 `loading`**：很多站点有长时间轮询或 WebSocket，导致 `wait_until="load"` 永不触发。换成 `domcontentloaded` 配合关键选择器等待更可靠。
3. **内网 SSRF 风险**：仅靠域名白名单不够，如果 DNS 被污染或目标站做反向代理到内网，仍然可能打到内部服务。可以额外在容器网络层做出口隔离，只允许访问公网 IP 段，或者强制走固定 HTTP 代理出口。
4. **动态内容加载时机**：部分站点把正文放在 `window.__INITIAL_STATE__` 里，页面 HTML 本身几乎是空壳。这种场景需要定制解析逻辑，无法通用化，但可以加一个 metadata 字段告诉 Agent 内容来源是否完整。
5. **token 估算粗糙**：字符数/4 的算法对中文误差大，会过早截断。如果英文为主可以接受，中文场景建议改用 `tiktoken` 精确计数。

## 可复用建议

- **分层设计**：将“请求层”“清洗层”“权限层”解耦，方便单独替换。比如以后想用 Scrapy 代替 Playwright，只需改请求层。
- **返回结构化错误**：别把原始异常堆栈抛给 Agent。返回 `{"error": "domain_not_allowed", "detail": "example.com not in whitelist"}` 能让 Agent 更理智地处理异常。
- **缓存友好**：对相同 URL 的内容可以短时间缓存（如 5 分钟），避免重复抓取同一页面迅速耗尽浏览器资源。缓存键带上是否截断、格式等参数。
- **可观测性**：记录每次抓取的耗时、状态码、最终文本大小、是否有截断，方便后期调优白名单和超时策略。
- **与 Agent 约定契约**：除了工具描述，还可以在系统提示中写“如果你抓取的页面内容被截断，请请求更小范围的 URL 或让用户提供具体段落”，让模型学会自纠正。

## 总结

Web Scraping 稽客并不是一个高深的技术组件，但它是连接 Agent 和开放互联网之间的第一道安全闸门。通过容器化无头浏览器、内容清洗与截断、域名白名单以及明确的错误契约，我们能把不可控的网页变成 Agent 可用的结构化知识，同时守住安全底线。在 OpenClaw 这类智能体框架中，这种“小工具”的工程化程度，往往决定了整个系统能不能从 Demo 走向持续运行的生产环境。

---

