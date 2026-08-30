---
title: Web scraping 稽客：给 Agent 的网页采集加一道安检
feedId: 35395
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景

Agent 做自动化时，网页是最不稳定的外部数据源之一。在 OpenClaw 工具链里，如果直接给模型一个裸的 HTTP 能力，模型很容易把网页当 API 用：同一域名短时间打爆、把整页 HTML 拖进上下文、误入登录页或内部系统，还会拿到一堆脚本、Cookie 横幅和跟踪参数。

## 问题

直接暴露“抓网页”的能力有三个工程风险：

1. **SSRF**：用户给了一个内网地址，Agent 当普通 URL 去抓。
2. **内容爆炸**：原始 HTML 单页可能几百 KB，进入上下文既浪费 token 又干扰推理。
3. **不可观测**：Agent 抓到什么、失败原因、是否命中 robots、最终跳转到哪里，没有记录。

## 做法：把“稽客”做成独立 MCP server

我会把网页采集封装成一个 `web_scrape` 工具，不让 Agent 直接接触底层 HTTP。核心流程固定：

1. 入参只收 URL，且必须是 http/https。
2. DNS 解析后校验 IP：拒绝 `127.0.0.0/8`、`10.0.0.0/8`、`172.16.0.0/12`、`192.168.0.0/16`、`169.254.0.0/16`、`::1` 等内网地址；每次 30x 跳转前重新校验。
3. 每个域名独立限速，默认 1 QPS；遇到 429/503 时读取 `Retry-After` 并做退避。
4. 请求前检查 `robots.txt`，返回 `robots_allowed` 和 `crawl_delay`，作为软约束。
5. 抓取后用 Readability 类算法提取正文，保留 `pre/code`，表格转成 GFM；删除 script、style、iframe、form、tracker。
6. 正文限制在 24k tokens 或 96KB 以内，截断时返回 `[truncated]` 和截断位置。
7. 统一返回元数据：`final_url`、`canonical`、`title`、`etag`、`last_modified`、`fetched_at`、`content_type`、`size`、`truncated`、`robots_allowed`、`sha256`。
8. 做短期缓存：按 `final_url + etag` 缓存，默认 10 分钟；有 `last_modified` 时优先条件请求。

Agent 看到的不是爬虫，而是一个会返回干净、受限、可追踪内容的工具。

## 踩坑点

- **DNS rebinding / 重定向绕过**：第一次解析是公网，第二个请求可能指向内网。所有 redirect 都要“解析 + 校验 IP + 再请求”，不能只在入口校验一次。
- **不要默认对每个站点开无头浏览器**：资源重、特征明显、容易被风控。先走纯 HTTP 抓取，遇到动态页面返回空正文时，明确报 `content is dynamic`，让 Agent 换方案。
- **Readability 会误删内容**：代码块、表格、公式常被当成噪声移除。提取时必须保留 `pre`，表格优先转 Markdown，不要直接丢文本。
- **robots.txt 解析别自己写**：通配符、Allow/Disallow 顺序、`*`、`$` 规则很容易出错，用成熟库处理。
- **巨型页面不是越全越好**：Agent 上下文有限，截断后必须返回截断位置，否则模型会拿半截内容下结论。
- **缓存键要包含规范化 URL**：去掉 `utm_*`、`fbclid` 等跟踪参数，避免同一内容重复抓取；但不能去掉会影响页面内容的参数。

## 可复用建议

- 把“稽客”做成独立 MCP server，而不是散落在 prompt 里的规则。规则会漂移，工具不会。
- 返回 `sha256` 和 `final_url`，让上层 Agent 有依据判断内容是否稳定。
- 对 `Content-Type` 非 `text/html` 的直接拒绝，避免 Agent 抓 PDF、压缩包后乱猜。
- 给一个 `allow_private` 开关，默认关闭；只有明确的内网自动化场景才打开。
- 日志至少保留：时间、Agent 会话、原始 URL、最终 URL、状态码、耗时、是否命中 robots、是否截断。出问题时能定位是站点问题还是 Agent 误用。

## 总结

不要让 Agent 直接拥有“抓网页”的原始能力，而是给它一个经过 URL 校验、限速、robots 检查、内容提取、截断和审计的 Web scraping 稽客。安全、干净、可观测，网页才会成为 Agent 的可靠数据源。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/4be7e3eef07d354b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/31e6fae78ddf2141.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/fb1dad2eea557894.png)

