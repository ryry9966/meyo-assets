---
title: Web scraping 稽客：让 Agent 安全地采集网页内容
feedId: 31476
source: 综合讨论
publishedAt: 2026-08-03
---

## 背景

Agent 的能力上限，很大程度上取决于它能不能拿到及时、准确的上下文。搜索引擎 API 要么限流要么付费，更多时候你要的只是一个具体页面的正文。很多人在给 OpenClaw 配「读网页」能力时，第一反应是写个简单的 fetch 工具，结果发现大多数站点只返回一个空壳，要么被风控拦掉，要么拿到的 HTML 没法用。

## 问题

裸抓页面的问题有三个层面：

- **渲染层**：页面是 JS 动态渲染的，纯 HTTP 请求拿不到正文；
- **风控层**：Cloudflare、UA 校验、频率检测，针对机房 IP 基本都是见一个封一个；
- **合规层**：不少站点 robots.txt 明确禁止抓取，但无脑采集器照样访问，既不礼貌也容易吃法律风险。

这不是换一个 HTTP 库能解决的，需要一套有策略、有降级预案的采集链路。

## 做法

我采用的方案是**分层降级**：

1. 第一层：带真实 UA 的 GET 请求 + Readability 提取正文，适合静态博客、文档站；
2. 第二层：走 Jina Reader（r.jina.ai）做 JS 渲染并转成 Markdown，适合绝大多数 SPA 和资讯站；
3. 第三层：本地起 Playwright 或 Firecrawl 服务，适合需要登录态或复杂交互的页面。

在 OpenClaw 里，我把它封装成一个独立插件，对外暴露三个工具：`fetch_url`、`fetch_with_reader`、`fetch_with_browser`。三者共用同一个缓存和限流队列，核心逻辑如下：

```python
key = hash(url + strategy)
if cache.get(key, ttl=24h): return cache[key]
wait(rate_limiter.acquire(domain))
html = fetch(url)
markdown = extract_content(html, max_chars=8000)
cache.set(key, markdown)
return markdown
```

关键约束：只保留正文 Markdown 和必要的元数据，原始 HTML 直接丢弃，避免撑爆 Agent 上下文；默认每域名限流 5 秒一个请求；UA 伪装成真实浏览器，并在注释里留下联系方式。

## 踩坑点

- **Cloudflare 无法保证突破**：Jina Reader 对部分站点也会失败。所以插件里必须做降级开关——第一层失败自动切第二层，再失败就返回缓存快照，或明确告诉 Agent「该页当前不可得」，让 Agent 自己决定下一步，而不是反复重试。
- **robots.txt 不是护身符**：有些站虽然允许爬虫，但对请求频率极端敏感。限流参数必须留出配置入口，不要写死。
- **编码坑**：不少老站点是 GBK，按 UTF-8 解析会乱码。建议统一用 charset-normalizer 做编码探测，再交给提取器。
- **上下文爆炸**：500KB 的 HTML 转成 Markdown 可能还剩 80KB，对 token 消耗非常大。我加了 `max_chars` 参数，默认截断到 8000 字符，并在文末注明「内容已截断」。

## 可复用建议

- **缓存是性价比最高的一层**。同一 URL 重复抓取，既浪费额度又容易被封，务必做持久化缓存（TTL 按站点类型区分，新闻类短、文档类长）。
- **优先 API 和 RSS**。抓取始终是兜底手段，不要把它当成默认方案。
- **维护「不可抓取」清单**：一旦某域名触发 403 或验证码，标记后短期内不再尝试，给插件留一个手动管理入口。
- **给插件加健康检查接口**：输出近 24 小时的成功率、平均耗时、封禁次数，方便持续调优。

## 总结

让 Agent 安全地采集网页，核心不是抓得快，而是抓得稳、抓得省。分层降级 + 缓存 + 限流 + 截断，这四件套能覆盖绝大多数实际场景。别指望一个工具通吃所有站点，把降级逻辑和失败反馈做扎实，Agent 才真正具备「读得动互联网」的可靠能力。

---

