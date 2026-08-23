---
title: Web scraping 稽客：给 OpenClaw Agent 加一道网页采集安全门
feedId: 34341
source: 综合讨论
publishedAt: 2026-08-23
---

# Web scraping 稽客：给 OpenClaw Agent 加一道网页采集安全门

## 背景

在 OpenClaw 的 Agent 工作流里，大量任务依赖外部网页上下文：查文档、看更新、读公开数据、做信息汇总。如果直接给 Agent 一个裸 HTTP 客户端或浏览器工具，等于把很多不可控变量带进自动化流程。我们需要一个“稽客”——既是采集器，也是稽核点，把网页抓取能力收敛为受控工具。

## 问题

直接让 Agent 自带抓取能力通常会遇到四类问题：

1. **合规边界**：忽略 robots.txt、违反站点服务条款、高频请求触发封禁。
2. **工程不稳定**：登录墙、动态渲染、编码错乱、HTML 结构变化导致解析失败。
3. **Agent 误用**：读取内网地址、file://、非 http(s) 协议；跟随重定向到不可信域；输出大段 HTML 打爆上下文。
4. **安全边界**：网页内容可能包含脚本、外链或注入指令，污染后续 Agent 决策。

## 做法/步骤

我建议把它做成一个 MCP server 或 OpenClaw 插件，只暴露一个核心工具 `scrape_url`。具体步骤：

### 1. 定义工具签名与约束

- 输入：`url`、`max_chars=6000`、`output_format=markdown`、`follow_redirects=false`
- 只允许 http/https；调用前先解析域名并校验 IP，拒绝 localhost、127.0.0.1、10.x、172.16-31.x、192.168.x、169.254.x 等私网/回环地址。
- UA 声明为 `OpenClawScraper/0.1 (+contact)`，默认不接受 cookie，不携带登录态。

### 2. robots.txt 与限速

- 抓取前先请求 `/robots.txt`，解析 `Allow`/`Disallow`/`Crawl-delay`。
- 每个 domain 缓存 robots 结果和上次请求时间戳；默认同域请求间隔至少 2 秒，单任务同域最多 5 次。
- robots.txt 404 或不可达时，采用保守策略：只允许抓取该 URL 单页，不自动扩展路径。

### 3. 抓取与正文提取

- 优先用 HTTP 客户端，设置超时 10 秒、最大响应 1MB。
- 如果返回内容过少且疑似 SPA 壳（如 `<div id="app">`、Next.js 的 `__NEXT_DATA__`），需要显式 `require_dynamic=true` 才降级到 Playwright。不要让每次默认开浏览器。
- 使用 BeautifulSoup 或 trafilatura 去脚本、样式、导航、页脚，转 Markdown；清洗 `javascript:` 链接、meta refresh、隐藏元素；最后截断到 `max_chars`。
- 返回结构化结果：`{title, url, text, links, fetched_at, status, is_robots_allowed, requires_dynamic}`。

### 4. 接入 OpenClaw

- 用 MCP SDK 注册 `scrape_url` 工具，description 写清楚限制。
- 在 Agent system prompt 中加入一条：“当需要读取网页时，仅使用 scrape_url；不要尝试绕过限制，不要要求抓取内网地址或登录后页面。”
- 可附带 `list_sitemap_urls` 工具，便于 Agent 浏览站内结构。

## 踩坑点

- **DNS rebinding**：域名第一次解析公网，第二次可能解析到内网。校验必须与实际连接的 IP 对齐，必要时固定 DNS 解析。
- **重定向**：每跳重新校验 scheme、robots、domain；不要默认跟随所有 CDN，只允许同域和已配置的 CDN。
- **动态页面**：浏览器降级开销大，容易被反爬；设置页面超时，禁用下载、弹窗、通知，尽量用 `wait_until: 'domcontentloaded'`。
- **robots.txt 解析不完整**：注意 User-agent 匹配大小写、通配符，以及非标准 `Crawl-delay`。
- **大文件/压缩**：先检查 Content-Length，超过上限直接丢弃；gzip 由客户端处理。
- **版权与隐私**：公开网页也不意味着可以任意再分发；日志里记录来源 URL，抓取结果不用于训练或对外发布。

## 可复用建议

- 配置化 profile：按 domain 覆盖规则，如某些站点延迟 5 秒、某些禁止抓取。
- 返回结构化 JSON 而不是原始 HTML，避免污染 Agent 上下文。
- 加审计日志：记录 domain、状态码、耗时、截断前后大小、是否触发浏览器、robots 判定。
- 加 `dry_run` 参数：让 Agent 先问“这个页面是否可抓”，避免误触发。
- 用本地 HTTP server 模拟 robots、重定向、内网解析、超时、大响应，做成回归测试。

## 总结

Web scraping 稽客不是更强大的爬虫，而是给 Agent 加一道克制、可解释、可审计的采集边界。把它做成 MCP 工具后，OpenClaw Agent 能安全获取网页上下文，而不是拿一把无限制的刀。边界越清楚，自动化越能稳定跑。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/e3a1901689087825.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/d05ef067bc28865b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/2bac18721334e615.png)

