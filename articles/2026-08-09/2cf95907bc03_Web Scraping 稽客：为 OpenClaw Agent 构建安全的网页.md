---
title: Web Scraping 稽客：为 OpenClaw Agent 构建安全的网页采集管道
feedId: 32254
source: 综合讨论
publishedAt: 2026-08-09
---

# Web Scraping 稽客：为 OpenClaw Agent 构建安全的网页采集管道

## 背景：当 Agent 开始阅读网页

OpenClaw 的 Agent 在自动化任务中常需要从公开网页提取信息——比如读取竞争对手的更新日志、汇总技术文档、或监听合作方站点变更。直接让 Agent 在运行时用 `curl` 或 `requests` 抓取是最简单的路径，但在生产环境中很快就会撞上三堵墙：

1. **反爬策略**：现代站点对无头流量极度敏感，IP 封禁、验证码、行为指纹检测层层叠加。
2. **状态一致性**：Agent 可能发起多次关联请求（例如先登录再查询），若没有持久的 Cookie/会话管理，每一步都可能失败。
3. **合规与可控**：无节制的抓取不仅影响目标服务，还可能让 Agent 陷入循环请求，消耗本地资源却没有产出有效信息。

「稽客」在这里指的不是某个工具，而是一种工程角色——负责**稽核与代理每一次网页访问**，确保安全、合规、可审计。我们需要设计一个面向 Agent 的采集中间层，把复杂度封装起来，对外只暴露高层语义接口。

## 问题拆解：Agent 视角的安全采集需求

通过梳理多个 OpenClaw 自动化实践项目，我把核心需求抽离成以下几点：

- **身份伪装**：可配置的 User-Agent、窗口尺寸、平台指纹，避免被识别为 Headless Chrome。
- **速率控制**：同域名请求间隔、并发上限、自动退避，必要时熔断。
- **会话隔离**：每个采集任务使用独立的浏览器上下文，Cookie/localStorage 互不串扰。
- **资源过滤**：屏蔽图片/字体/CSS/分析脚本，加速渲染并可降低特征暴露。
- **错误透传与重试**：遇到 403/503 或验证码时，向 Agent 返回结构化错误而非原始 HTML。

在 OpenClaw 生态中，天然适合用 **MCP (Model Context Protocol)** 承载这些能力。MCP Server 运行在 Agent 侧，Agent 通过标准化工具调用发起采集，Server 内部驱动浏览器完成请求与解析。

## 实践步骤：构建安全的 Scraping MCP Server

以下方案基于 Playwright + Python，但同样适用于 Puppeteer/Playwright 的 TypeScript 实现。步骤按工程推进顺序给出。

### 1. 初始化 MCP Server 骨架

```bash
mkdir mcp-scraping-kecher && cd mcp-scraping-kecher
python -m venv .venv && source .venv/bin/activate
pip install mcp playwright beautifulsoup4
playwright install chromium
```

新建 `server.py`，注册两个基础工具：

- `fetch_page(url, wait_for_selector?)`：返回页面纯文本、标题、关键元信息。
- `capture_snapshot(url)`：返回截图 base64（用于 Agent 视觉判别，如验证码处理）。

### 2. 添加安全层——浏览器上下文工厂

```python
from playwright.async_api import async_playwright
import asyncio, random

class SecureBrowser:
    def __init__(self, config):
        self.config = config  # 包含 user_agent, proxy, delays 等

    async def start(self):
        self.pw = await async_playwright().start()
        self.browser = await self.pw.chromium.launch(
            headless=True,
            args=[
                "--disable-blink-features=AutomationControlled",
                "--no-sandbox",
                f"--user-agent={self.config['user_agent']}",
            ]
        )

    async def new_context(self):
        context = await self.browser.new_context(
            viewport={"width": 1440, "height": 900},
            locale="en-US",
            timezone_id="America/Chicago",
            permissions=[],
            extra_http_headers={
                "Accept-Language": "en-US,en;q=0.9"
            }
        )
        # 注入反检测脚本
        await context.add_init_script("""
            Object.defineProperty(navigator, 'webdriver', { get: () => false });
        """)
        return context
```

**要点**：每次 `fetch_page` 调用都使用 `new_context()`，用完立即关闭，实现请求级隔离，避免 Cookie 累积导致的指纹漂移。

### 3. 速率控制与域名监管

在 MCP 工具调用入口增加一个基于域名的令牌桶：

```python
from collections import defaultdict
import time

class RateLimiter:
    def __init__(self, max_per_second=2):
        self.tokens = defaultdict(lambda: (max_per_second, time.time()))

    async def acquire(self, domain):
        limit, last = self.tokens[domain]
        now = time.time()
        if now - last >= 1.0:
            self.tokens[domain] = (limit, now)
            return
        wait = 1.0 - (now - last)
        await asyncio.sleep(wait)
        self.tokens[domain] = (limit, time.time())
```

同时维护一个机器人排除协议（robots.txt）缓存，每次对陌生域名的首次访问先检查允许路径，`disallow` 的直接拒绝并返回给 Agent。

### 4. 结构化输出与错误信令

不要直接把 HTML 丢给 Agent，而是用 BeautifulSoup 提取主内容，返回 Markdown 或纯文本。若遇到 403，返回：

```json
{
  "error": "blocked",
  "status_code": 403,
  "hint": "The server returned 403. Consider adding proxy rotation or check if the target requires login."
}
```

Agent 收到结构化错误后可以触发备选路径（例如切换出口 IP 或通知人工）。

### 5. 集成到 OpenClaw

OpenClaw 的 MCP 配置中直接指向本服务：

```yaml
mcp_servers:
  scraping-kecher:
    command: "python"
    args: ["path/to/server.py"]
```

Agent 的 System Prompt 中加入使用指引即可，例如：“当需要获取网页信息时，优先使用 `fetch_page` 工具；如果返回 `blocked` 错误，请考虑等待后重试或请求人工介入。”

## 踩坑记录

- **无头浏览器检测绕过不彻底**：仅修改 `navigator.webdriver` 不够，高阶反爬会检测 `chrome.runtime`、`navigator.plugins.length` 等。在 context 初始化时注入更完整的 stealth 脚本，或直接使用 `playwright-stealth` 库可显著提升通过率。
- **资源加载耗时而无用**：务必在 context 中 route 拦截图片、字体、分析脚本（`route('**/*', lambda r: r.abort() if r.request.resource_type in ['image','font','media','stylesheet'] else r.continue_())`），单个页面加载时间可从 8s 降至 2s 以内。
- **内存泄漏**：忘记关闭 context 或 browser 导致 Chromium 子进程残留。使用 `try...finally` 确保清理，或在 MCP Server 的 `shutdown` 生命周期钩子中关闭全局 browser 实例。
- **动态内容等待不稳定**：`wait_for_selector` 需结合超时和 fallback，例如等待网络空闲（`wait_for_load_state('networkidle')`）作为兜底。

## 可复用建议

1. **配置外移**：将 User-Agent 列表、代理地址、延时范围写入 YAML/JSON 配置文件，方便不同 Agent 任务切换身份。
2. **审计日志**：记录每次抓取的时间戳、目标 URL、状态码、耗时、响应大小，便于排查问题和生成使用报告。
3. **Cache 层**：对不变内容（例如静态文档）可增加基于 URL 的 TTL 缓存，通过工具参数 `use_cache=True` 控制，减少重复请求。
4. **通用 MCP 模板**：将上述 Security & Rate-Limit 逻辑抽成基类，以后添加新采集工具（如 `extract_table`、`monitor_feed`）只需继承并关注解析逻辑。

## 总结

给 Agent 配一个网页采集“稽客”层，本质上是通过 MCP 将不定向的网络风险收敛为可控、可审计的标准化接口。OpenClaw 的用户只需花一次搭建成本，之后 Agent 在需要网页信息时就能遵循同一套安全规则，不会因为一次“手滑”的全量抓取导致出口 IP 进入黑名单。该方案同样适用于内部 Wiki、需要登录的 SaaS 面板等受限环境，只需扩展会话预置逻辑即可。

当下 Agent 采集不仅是技术问题，更是一种资源契约——我们希望对目标服务保持尊重，同时保障自动化流程的健壮。这套“稽客”思路，希望对有同类需求的小伙伴有所启发。

---

