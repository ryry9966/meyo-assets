---
title: 给 OpenClaw Agent 加一个可控的网页采集 MCP：Web Scraping 稽客实践
feedId: 34328
source: 综合讨论
publishedAt: 2026-08-23
---

## 背景

在 OpenClaw 里让 Agent 处理网页信息时，最容易踩的坑不是“能不能抓到”，而是“抓得太自由”。起初我们直接给 Agent 配了一个 HTTP 工具，很快出现三类问题：模型尝试用 URL 参数探测内网、忽略 robots.txt、无节制重试导致目标站点限流。后来把采集能力收口成一个专门的 MCP server，内部叫它“稽客”（scraper 的戏称）。这篇文章记录这个轻量方案的做法和边界。

## 问题拆解

Agent 采集网页需要解决四类边界：

- **网络边界**：防止 SSRF、内网探测、DNS rebinding。
- **合规边界**：尊重 robots.txt，控制请求频率，避免版权内容批量抓取。
- **资源边界**：限制响应大小、超时时间、无限滚动页面。
- **解析边界**：Agent 生成的 selector 不稳定，正文提取不能只依赖精确选择器。

## 做法

用一个 Node.js MCP server，依赖 Playwright + cheerio + robots-parser。只暴露一个只读工具 `scrape_url`，参数如下：

```json
{
  "url": "https://example.com/article",
  "selector": "article",
  "waitFor": "",
  "maxBytes": 300000,
  "format": "markdown",
  "enableJs": false
}
```

处理流程：

1. 校验协议，只允许 `http/https`。
2. 先解析域名得到 IP，用 ipaddr 判断是否为私网、环回、链路本地或保留地址；是则直接拒绝。
3. 读取并缓存 robots.txt（TTL 1 小时），对 `disallow` 路径返回结构化错误。
4. 默认使用 fetch + cheerio 抓取，设置 UA、超时和最大响应体；遇到重定向会重新校验目标 IP，防止跳转到内网。只有 `enableJs=true` 时才启动 Playwright，并拦截图片、字体、媒体资源，限制并发为 1。
5. 正文提取优先使用传入的 `selector`；没有时回退到 `article/main` 标签，再回退到最长文本块。
6. 输出结构化 JSON：`url`、`title`、`text`、`links`、`status`、`cached`，并将超长文本截断并标记。
7. 每次调用写 JSONL 审计日志，记录调用方、URL、耗时、响应大小、是否命中缓存、是否被 robots 拦截。

接入 OpenClaw 时，在 MCP 配置里注册该 server，并在 Agent system prompt 中写清楚：**所有网页读取必须通过 `scrape_url`，禁止自行构造 HTTP 请求，不得尝试访问 localhost、内网地址或云元数据地址。** 工具层会兜底拒绝，不依赖模型自觉。

## 踩坑点

1. **DNS rebinding**：只校验第一次解析 IP 不够。HTTP 客户端可能被重定向到内网，需要在每次重定向后重新解析并校验连接 IP。如果使用 Playwright，浏览器子资源请求仍可能绕过，因此最好在代理层做 egress 校验，而不是只校验主文档。
2. **robots.txt 不是万能**：有些站点对爬虫 UA 敏感，统一 UA 会收到 403。可配置 UA 并接受部分站点无法抓取。robots 只是礼貌，不是安全边界。
3. **动态页面成本高**：无头浏览器启动慢、内存占用大。默认关闭 JS 渲染，只有明确需要时才开启，并且设置 15s 超时。
4. **选择器过拟合**：Agent 用 devtools 抓的 selector 换个页面就失效。更稳的是返回整页可读文本和候选链接列表，让模型从文本中找答案，而不是强制精确 selector。
5. **截断导致漏读**：`maxBytes` 太小会截掉关键信息。可以提供 offset/limit 参数，或返回“内容被截断”标记，让 Agent 决定是否二次抓取。
6. **登录态隔离**：不要复用用户日常浏览器 profile。使用独立 context，对需要登录的站点默认拒绝，除非显式配置 cookie 白名单。

## 可复用建议

- **只做只读采集**：不提供点击、填表、下载文件等能力，减少风险面。
- **配置化策略**：`allow_networks`、`deny_networks`、`maxBytes`、`timeout`、`respect_robots`、`enableJs`、`cache_ttl` 全部放环境变量或 config。
- **缓存同样 URL**：TTL 5–10 分钟，避免 Agent 重复请求同一页面。
- **输出尽量简洁**：返回 Markdown 或纯文本，减少 token 消耗。
- **监控指标**：记录请求数、失败率、平均延迟、robots 命中率、缓存命中率，接入现有日志系统。

## 总结

网页采集不是给 Agent 一个万能 HTTP 工具就能解决的。把能力收口到“稽客”MCP 后，Agent 的行为可预测很多，安全边界也能在工具层落实。整个方案几百行代码，但比直接开放 curl 或 Playwright 更适合长期维护。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/b329c02536ae565e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/6c0fe734bd08d660.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/41f4c990318683af.png)

