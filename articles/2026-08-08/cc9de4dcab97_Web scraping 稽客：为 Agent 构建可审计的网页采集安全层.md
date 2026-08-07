---
title: Web scraping 稽客：为 Agent 构建可审计的网页采集安全层
feedId: 32058
source: 综合讨论
publishedAt: 2026-08-08
---

## 背景

在 OpenClaw 的自动化工作流中，Agent 常被要求“查一下这个网页的最新信息”“提取某论坛的帖子列表”等。如果让 Agent 直接调用原生的 HTTP 客户端或无头浏览器，很快就会踩坑：IP 被反爬封禁、忽略 robots.txt 引发合规争议、请求风暴拖垮目标站点，甚至无意中采集到恶意脚本或敏感数据。这些问题在长时间运行的自治 Agent 中会被放大——你不可能手动审查每一次页面抓取。

因此我们迫切需要一层可嵌入的**稽客（Scraping Auditor）中间件**，让每一次网页采集都经过策略检查、速率控制和内容净化，同时留下可追溯的审计日志。它在 Agent 与外部 Web 之间充当守门人，既保障采集行为的安全可控，又不妨碍任务的正常执行。

## 问题拆解

一个无约束的 Agent 在采集网页时通常会暴露以下四类风险：

1. **合规与法律风险**：无视 robots.txt，频繁抓取受保护路径，可能被视为恶意爬虫。
2. **资源滥用与反爬**：短时间大量请求导致 IP 被封，目标站点响应 429/403，任务中断。
3. **内容安全**：原始 HTML 中混杂的 `<script>`、钓鱼链接或隐私信息（如邮箱、手机号）可能进入 Agent 的记忆或输出。
4. **可观测性缺失**：Agent 到底抓了哪些 URL、返回了什么状态码、是否被重定向，完全黑盒，排查问题困难。

传统做法是在 Agent 的提示里加一句“请尊重 robots.txt 并控制频率”，但这靠自律的方式几乎无效。我们需要一个工程化的稽核层，将策略内置进每一次 HTTP 请求的链路中。

## 设计方案：稽客中间件

我们将稽客实现为一个独立的服务（可部署为 MCP server），对外暴露一个 `fetch_page` 工具。Agent 不再直接发起 HTTP 请求，而是通过该工具经稽客中转到目标网页。架构上依次经过四个模块：

### 1. 策略检查层（Policy Checker）

首先解析目标域名的 `robots.txt`。为避免每次请求都去抓取，会本地缓存解析结果并遵守 `Crawl-delay`、`Allow`/`Disallow` 规则。同时维护一个可配置的域名白名单/黑名单，以及 URL 路径模式匹配。

配置示例（`auditor-policy.yaml`）：

```yaml
allowed_domains:
  - "docs.example.com"
  - "api.github.com"
blocked_patterns:
  - "*/admin/*"
  - "*.pdf"
default_crawl_delay: 3  # 秒
user_agent: "OpenClaw-Auditor/1.0"
```

若请求被策略拦截，直接返回拒绝原因（如 “Blocked by robots.txt: Disallow /private”），Agent 可据此调整行为而不是傻等。

### 2. 速率控制与请求安全（Rate Limiter & Safe Fetcher）

对每个域名使用令牌桶算法，严格按照 `Crawl-delay` 发放令牌。同一域名的并发请求数也被限制。底层使用 `httpx`（静态页面）或按需调用 Playwright（需要 JS 渲染的场景），并设置超时、自动重试（带指数退避）、User-Agent 轮换池和合理的 TLS 指纹伪装。对 429 响应，退避时间参考 `Retry-After` 头。

### 3. 内容净化（Content Sanitizer）

收到响应后，先剥离所有 `<script>`、`<style>`、`<iframe>` 标签，移除注释和多余空白。接着用正则过滤常见的隐私模式（邮箱、手机号、IP 地址），除非策略显式允许（如已知的公共数据）。对于 JSON 响应同样做字段黑名单过滤。净化后的纯文本或结构化数据再返回给 Agent。

### 4. 审计日志（Auditor Log）

每次请求记录：时间戳、请求 URL、域名、状态码、是否命中缓存、耗时、被策略拦截原因、内容大小。日志可输出到本地文件或集中式日志系统，便于后续分析 Agent 的行为模式，例如是否频繁重试被拒的 URL。

将上述流程封装成一个 MCP 工具，挂载到 OpenClaw 的 function call 列表即可：

```python
# 伪代码：MCP 工具入口
async def fetch_page(url: str, render_js: bool = False) -> dict:
    result = await auditor.process(url, render_js)
    return {"content": result.clean_text, "status": result.http_status, 
            "from_cache": result.cached}
```

## 踩坑记录

在实际部署中，有几个容易翻车的地方：

- **robots.txt 通配符解析不标准**：许多库对 `*` 和 `$` 的处理存在差异，最好参考 Google 的解析规范自实现或使用经过验证的解析器。尤其注意路径中是否允许空路径 `/` 被 Disallow。
- **速率控制导致任务超时**：如果 Agent 需要连续抓取多个页面，而 `Crawl-delay` 为 10 秒，整个任务流可能超时。此时可引入“请求队列 + 异步通知”模式，让 Agent 以非阻塞方式等待结果，或在设计任务时要求减少不必要的抓取。
- **内容净化误伤结构化数据**：有些网站用 `<script type="application/ld+json">` 存放有效元数据，如果粗暴删除所有 script 标签会丢失信息。因此净化规则需要支持例外配置，按 type 或 data-* 属性保留。
- **动态渲染成本**：默认走静态 HTTP 请求可以覆盖大多数场景，但遇到 SPA 页面会返回空壳。为控制成本，可根据 URL 模式或响应内容长度触发动态渲染，并缓存渲染结果。

## 可复用建议

- **封装为 MCP 服务**：将稽客连同策略文件一起打包，一份配置即可服务多个 Agent。OpenClaw 只需配置一个 external tool，不必在每个工作流里重复造轮子。
- **缓存层**：对于不常变化的页面（如文档、API 规范），引入可配置 TTL 的响应缓存，能显著降低请求频率和延迟。
- **配置热更新**：通过文件监听或管理接口，在线修改域名白名单或速率参数，无需重启服务。
- **导出审计数据**：将日志结构化为 JSON，导入到 Grafana 等工具做可视化，能快速发现哪些目标站点不稳定、哪些规则误拦截。

## 总结

让 Agent 自由采集网页是危险的，但完全禁止又会限制其能力。稽客提供了一条工程化的中间路径：通过策略检查、速率控制、内容净化和审计日志四个环节，将安全策略注入每一次请求。对于 OpenClaw 的用户来说，只需一个可控的 `fetch_page` 工具，就能在不牺牲效率的前提下，大幅降低 Agent 运行中的法律、安全和运维风险。这套模式不仅适用于 OpenClaw，同样可扩展至任何需要 Web 访问的自治 Agent 系统。

---

