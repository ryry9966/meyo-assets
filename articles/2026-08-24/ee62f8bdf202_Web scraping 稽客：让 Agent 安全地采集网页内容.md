---
title: Web scraping 稽客：让 Agent 安全地采集网页内容
feedId: 34549
source: 综合讨论
publishedAt: 2026-08-24
---

## 背景

在 OpenClaw、Agent、MCP 这类自动化流程里，网页采集几乎是最常见需求。无论是给 Agent 补充搜索结果、采集文档页，还是监控价格变化，都需要一个能安全访问外部网页的采集层。这里把它叫“稽客”——不是通用爬虫，而是带稽核、有边界、输出结构化的轻量抓取工具。

很多实践里，Agent 被直接赋予执行 Python/Node 脚本、调用 requests 或 Playwright 的能力，这带来几个典型问题：网络边界失控、解析结果不稳定、触发反爬、日志不可审计。

## 问题

直接让 Agent 执行任意抓取脚本，相当于把“访问权限”交给了模型。可能出现 SSRF（访问内网/云元数据）、无限制下载大文件、频率不可控、Cookie 或登录态泄露、页面改版后选择器失效、输出非结构化文本难以被后续节点消费。尤其在接 MCP/插件后，如果一个 scrape tool 没有做好边界，Agent 的一句“帮我抓这个网址”就可能变成安全事件。

## 做法/步骤

1. **分离抓取与解析**  
   抓取负责拿到 HTML/最终 DOM，解析负责抽取字段。两层解耦，避免一个函数既发请求又做选择器，也便于替换不同后端。

2. **网络边界前置**  
   只允许 http/https；目标域名解析后检查 IP，拒绝私有/保留地址、本地回环、云元数据地址；限制重定向次数和跨域重定向；设置连接超时、总超时、最大响应体。

3. **动态渲染按需启用**  
   优先用轻量 HTTP 客户端 + HTML 解析；仅在需要 JS 渲染时启用 Playwright，并设置资源屏蔽（图片、字体、媒体）、禁用下载、等待特定 selector 或固定时间。

4. **限速与缓存**  
   同域 QPS 限制，同 URL 哈希缓存，失败退避。避免把对方的反爬策略打成封禁。

5. **解析层结构化**  
   优先用可读性算法或用户提供的 JSON Schema / CSS selector；使用 LLM 抽取时，要求输出严格 JSON，并用 schema 和样例校验。

6. **输出统一**  
   每个 scrape 工具返回固定结构，包含 `title`、`url`、`final_url`、`status_code`、`elapsed`、`content_type`、`size`、`error`、`content` 或 `extracted` 字段。

7. **以 MCP server 暴露**  
   实现 `scrape_url`、`extract_schema`、`render_page`、`check_robots` 等工具，让 Agent 只能通过这些工具访问网页，而不是执行任意代码。

## 踩坑点

- **SSRF 绕过**：必须同时处理 `127.0.0.1`、`localhost`、`0.0.0.0`、`169.254.169.254`、IPv6、十进制/十六进制 IP、DNS rebinding。最佳实践是解析后对每个 A/AAAA 记录做校验，再连接。
- **robots.txt**：公开数据也要遵守规则，可以在工具中先请求 robots.txt 并解析 allow/disallow，必要时拒绝。
- **动态页面等待**：`networkidle` 不可靠，容易卡住或过早返回。建议用 `wait_for_selector` 加超时，或等待网络静默与关键元素组合。
- **内容编码和大体积**：注意 gzip/br 解码；用 `Content-Length` 提前判断，限制最大读取字节数，用流式截断而不是全部加载。
- **选择器脆弱**：不要写又长又脆的 CSS path；优先使用 `data-testid`、`id` 或语义化结构；提供 fallback 提取。
- **LLM 解析幻觉**：如果让 LLM 从 HTML 抽取字段，返回可能是“大概”格式，需要 schema 校验、类型检查、空值兜底。

## 可复用建议

- 把 scraper 封装成带配置的 MCP server，提供域名白名单/黑名单、QPS、超时、UA 设置。
- 所有请求记录 `final_url`、状态、耗时、响应体哈希，方便排障和审计。
- 提取输出使用 JSON Schema，而非自然语言描述，方便 Agent 后续调用工具。
- 对失败重试采用指数退避，避免在对方网站故障时造成雪崩。
- 如果站点明确禁止自动化，不要对抗反爬，转人工或换数据源。

## 总结

“安全地采集”不是让 Agent 变得更聪明，而是把采集能力封装成有边界、可观测、可复用的工具。边界决定 Agent 能做什么，结构决定后续流程能消费什么，日志决定出问题时能查什么。这三点比单纯追求抓取成功率更重要。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/1c4ab9dd302a08dc.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/1a90bbcf1cbb2c08.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/470d13d0483bc282.png)

