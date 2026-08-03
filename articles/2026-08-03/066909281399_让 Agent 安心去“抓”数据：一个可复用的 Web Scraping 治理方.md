---
title: 让 Agent 安心去“抓”数据：一个可复用的 Web Scraping 治理方案
feedId: 31502
source: 综合讨论
publishedAt: 2026-08-03
---

## 背景

最近在帮社区成员调一个 OpenClaw 的自动化任务：让 Agent 定时去某个行业站点抓取竞品公告，然后汇总成本地 Markdown 让下游 Agent 做摘要。任务本身不复杂，但跑了两天就出了问题：IP 被封、抓回来的内容里全是登录弹窗、偶尔还会把整个 HTML 塞给大模型导致 token 烧得飞快。

这类问题在 Agent 自动化场景里其实非常普遍。本地 Agent 不像在线服务那样有成熟的分布式爬虫基础设施，一个循环失控就能把站点打死。今天的帖子不聊反爬对抗，只聊如何让 Agent 在“安全、克制、可维护”的前提下，把网页内容干净地取回来。

## 问题拆解

Web scraping 对 Agent 来说有三个核心痛点：

1. **被反爬误伤**：Agent 的 headless 浏览器默认指纹太明显，触发验证码或 403，影响整个自动化链路。
2. **抓回来的东西“脏”**：HTML 里混着导航、弹窗、脚本，直接喂给模型既费 token 又干扰判断。
3. **失控风险**：Agent 如果没被约束，可能在不该快速请求的时候疯狂刷页面，把自己搞成攻击源。

## 做法与步骤

我目前验证过的一套方案是“双层抓取 + 语义抽提 + 权限护栏”，实测在 OpenClaw 环境下可以稳定跑一周以上不封 IP。

**第一步：把抓取逻辑从 Agent 决策里剥出来**

不要让 Agent 直接拿着工具去抓 URL，而是给它暴露一个固定的 MCP 工具，比如 `fetch_article(url)`。这个工具内部封装好了所有抓取细节：请求头伪装、超时控制、重试、限速。

实现上我用的是 Python 的 `playwright` 加 stealth 插件，保持 headless 模式但注入合理的 UA 和浏览行为。核心代码大概长这样：

```python
from playwright.sync_api import sync_playwright
from playwright_stealth import stealth_sync

def fetch_clean(url: str) -> dict:
    with sync_playwright() as p:
        browser = p.chromium.launch(headless=True)
        ctx = browser.new_context(
            user_agent="Mozilla/5.0 (Windows NT 10.0; Win64; x64) ...",
            viewport={"width": 1280, "height": 800},
        )
        page = ctx.new_page()
        stealth_sync(page)
        page.goto(url, timeout=15000, wait_until="domcontentloaded")
        page.wait_for_selector("article", timeout=5000)  # 等关键内容
        html = page.content()
        browser.close()
        return {"url": url, "html": html}
```

**第二步：内容抽提，只把“正文”交给 Agent**

抓回 HTML 后不要直接返回给 Agent。用 `trafilatura` 或 `readability` 做正文提取，把标题、作者、发布时间和纯文本段落单独拆出来，转成结构化 dict。

```python
import trafilatura

def extract(html: str) -> dict:
    doc = trafilatura.extract(html, output_format="json", with_metadata=True)
    return json.loads(doc)
```

这一步的收益是 token 消耗直接降了几个量级——原来一篇文章 2 万 token 的 HTML，提取后只剩 2000。

**第三步：给 Agent 套上“节流护栏”**

这是最容易忽略但最重要的一步。我在 MCP 服务里加了全局 token bucket：

```python
import time
from threading import Lock

class Throttle:
    def __init__(self, rate: float):
        self.interval = 1.0 / rate
        self.last = time.monotonic()
        self.lock = Lock()

    def wait(self):
        with self.lock:
            now = time.monotonic()
            delta = self.interval - (now - self.last)
            if delta > 0:
                time.sleep(delta)
            self.last = time.monotonic()
```

默认限速是每 5 秒一个请求，同时把单次任务的最大抓取页数限制在 50 页以内。Agent 超限会直接收到“限流中，请稍后再试”的返回，不会真的发出去。

**第四步：落地到 OpenClaw 的 MCP 配置**

把上面的服务跑成本地 HTTP 或 stdio server，然后在 OpenClaw 的工具配置里挂载。这样 Agent 决策时只认 `fetch_article` 这个语义工具，完全不接触底层网络细节，行为更可预测，也更好审计。

## 踩坑点

几个实际踩过的坑，拿出来分享一下：

- **wait_until 参数很关键**：默认 `load` 事件要等所有资源加载完，某些站点会挂 30 秒以上，改成 `domcontentloaded` 再手动等要快得多。
- **个别站点反爬不在 UA 在 TLS 指纹**：这种情况 Python requests 完全没戏，Playwright 反而能过，因为它的 TLS 指纹和真实 Chrome 一致。
- **弹窗劫持**：很多站点有 cookie 同意弹窗或“打开 App”浮层，直接抓正文会失败。我的方案是准备几组点击选择器：`.cmpboxbtn, #consent-accept, .close-btn` 等，先尝试点击再提取。
- **移动端页面可能是捷径**：如果桌面版反爬严格，试试给 Playwright 设移动设备的 UA 和 viewport，有些站对移动端几乎不设防，而且 M 站页面通常更干净。

## 可复用建议

1. **抓取服务独立成组件**：别把抓取写死在 Agent 里，做成一个独立 MCP 服务，换站点只需改规则不用改 Agent 逻辑。
2. **一律走结构化提取**：HTML 直接喂大模型是灾难，先抽提再交付，哪怕多一层转换。
3. **限速写死成产品策略**：不做运行时可调，避免 Agent 钻空子。
4. **留好 HTTP 状态码与错误返回**：让 Agent 学会从“403”里读懂需要降低频率，而不是盲目重试。
5. **做好缓存**：同一个 URL 一天内只抓一次，存成本地 JSON，能省 90% 的重复流量。

## 总结

Web scraping 对 Agent 来说早已是刚需，但它本质上是“用工程手段对抗不确定性”的过程。把抓取环节从 Agent 的自主决策中剥离出来，用专门的 MCP 服务去治理流量、提取内容和控制风险，才能让 Agent 真正做到“安全地采集”。

这套方案不依赖任何商业服务，所有组件都是开源软件，搬到社区里可以立刻复用。如果大家在实践中碰到更刁钻的站点，欢迎在评论区把 case 丢出来，我们继续把这块的玩法做扎实。

工程化的核心不是技术多炫，而是让系统在无人值守时也能稳定运行——这正是 Agent 程序比人更需要的品质。

---

