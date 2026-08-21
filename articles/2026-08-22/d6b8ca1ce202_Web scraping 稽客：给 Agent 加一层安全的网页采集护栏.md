---
title: Web scraping 稽客：给 Agent 加一层安全的网页采集护栏
feedId: 34100
source: 综合讨论
publishedAt: 2026-08-22
---

最近给 OpenClaw 里的 Agent 接网页采集，发现最大的问题不是“能不能爬到”，而是“爬得是否安全、是否可维护”。Agent 的行为很容易失控：遇到链接就抓、重试毫无节制、不检查 robots、选择器写死，最后 IP 被封或数据漂移。于是我把抓取能力收敛成一个带稽核层的 MCP server，让 Agent 只通过受控工具访问网页。

## 背景与问题

Agent 需要从网页提取信息，例如商品页、文档站、公告列表。常见做法是让 Agent 直接执行 `requests.get()` + `BeautifulSoup`，小规模还能用，稍微上量就出问题：

- 不检查 robots.txt，可能抓了禁止路径。
- 请求头过于明显，Python 默认 UA/TLS 指纹容易被识别。
- 无并发限制与退避，429 后继续打。
- 选择器依赖 HTML 结构，页面一改就失效。
- 动态页面静态拿不到，直接上 headless 成本高且易触发风控。

这些最终不是模型能力问题，而是抓取基础设施缺失。

## 做法与步骤

我把能力封装成三个 MCP 工具：`check_robots`、`scrape_text`、`scrape_structured`。Agent 调工具时，必须经过统一稽核层，而不是绕过。

1. **预检 robots 与站点规则**  
   每个站点首次访问前拉取 robots.txt，解析 User-agent、Allow/Disallow、Crawl-delay、Sitemap。结果按域名缓存，默认尊重 Crawl-delay。路径匹配要处理 `*` 和 `$` 规则，解析失败时采用保守策略：只允许抓首页或直接拒绝进入动态渲染。

2. **请求侧限制**  
   统一请求头，UA 使用可辨识的真实浏览器版本，并补齐 Accept、Accept-Language、Sec-Fetch 等。HTTP 客户端优先用 `curl_cffi` 或 Playwright 的 fetch 能力，降低 TLS 指纹差异。超时设为 10s，重试仅针对 429/5xx 和连接错误，指数退避并加随机抖动。并发按域名限制，默认 1-2 QPS。

3. **提取与结构校验**  
   优先解析 JSON-LD、meta、可读性正文，其次才是配置化 CSS selector。选择器放在站点配置里，不硬编码在代码中。提取后校验关键字段是否为空，计算内容哈希，发现字段缺失或哈希剧变时返回“内容结构可能变化”，而不是把脏数据丢给 Agent。

4. **动态渲染的触发条件**  
   默认先静态抓取。只有在 robots 允许、静态结果缺失目标字段、且站点配置里开启 `render: true` 时，才启动 headless。渲染时禁用图片、CSS、字体，限制渲染预算（如 5s），降低资源消耗和风控概率。

5. **审计与熔断**  
   每次抓取记录：url、robots 决策、状态码、耗时、字段哈希、是否命中风控。连续 429 或 block 时自动熔断该域名，返回明确错误给 Agent，而不是无限重试。

## 踩坑点

- **robots.txt 解析不能只靠库**，很多站点格式不规范。需要额外处理 BOM、注释、大小写、sitemap 中的相对路径。
- **TLS/HTTP2 指纹比 UA 更重要**。Python requests 经常在 Cloudflare 站点直接 403，换个 UA 没用，要升级到 `curl_cffi` 或 Playwright。
- **重试风暴是自找的**。404/403/410 不要重试，429 要看 Retry-After，5xx 最多重试 2 次。Agent 容易把“再试一次”当作万能解法，必须在工具层拦截。
- **动态渲染不是万能的**。登录墙、验证码、地区限制、反爬 JS 挑战可能无法通过，需要识别并快速失败，让 Agent 选择其他来源或人工处理。
- **选择器漂移会导致数据污染**。建议对关键字段配置多个备选 selector，并做非空校验，避免把“页面改版提示”当正文提取。

## 可复用建议

- **站点配置 YAML 化**：每个站点一个配置，包含 selectors、rate_limit、render、allowed_paths、headers 等。不要把站点逻辑散落在 Prompt 或代码里。
- **缓存降级**：对同一 URL 在 TTL 内直接返回缓存，既能降低站点压力，也能在站点抖动时提供兜底。
- **提供“预检”工具给 Agent**：让 Agent 抓取前先问 `check_robots`，把决策权留在工具层，避免模型自行判断。
- **记录采样内容**：每次抓取保存前 2KB 文本摘要，方便排障时判断是否被风控页污染。
- **设置全局 kill switch**：当某域名熔断后，后续请求直接返回结构化错误，不再进入重试队列。

## 总结

安全地采集网页，不是教 Agent 绕过反爬，而是把合规检查、频率控制、结构校验、审计熔断做成默认能力。Agent 只需要描述“要什么”，基础设施负责安全地拿回来。这样既能降低封禁风险，也能让抓取结果更可靠、可维护。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/c010122af9859184.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/b6f06b8e6b15c7d2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/227c98fdb5524fc3.png)

