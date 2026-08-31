---
title: Web scraping 稽客：让 Agent 安全地采集网页内容
feedId: 35541
source: 综合讨论
publishedAt: 2026-08-31
---

# Web scraping 稽客：让 Agent 安全地采集网页内容

## 背景

在 OpenClaw 或 MCP 插件里，Agent 经常需要读网页：查文档、做监控、取接口字段。但直接把 `requests` 或 Playwright 暴露给模型很危险：可能访问内网、拉回几 MB HTML 打爆上下文、带出登录态。

所以我把网页读取收口成一个小工具，叫“稽客”。它不追求爬虫能力，只做一件事：在 Agent 和外部网页之间加一层默认拒绝、内容最小化、可审计的边界。

## 问题

主要四个坑：

- SSRF：模型会尝试 `169.254.169.254`、`localhost`、内网 IP。
- 上下文膨胀：普通页面 HTML 1-2MB，直接给模型贵且容易丢重点。
- 隐私泄漏：自动复用 Cookie、系统代理或浏览器配置。
- 不可复现：同一 URL 每次抓，内容变化无法回溯，也容易被限流。

## 做法

在 OpenClaw 里用 MCP 暴露 `fetch_web_page(url, max_chars, extract_mode)`，流程如下：

1. 只允许 http/https，禁止 file、ftp、IP 直连。
2. 域名解析结果需通过内网校验；重定向每跳都重新校验。
3. GET 请求，自定义 UA，超时 8 秒，最大响应 2MB；不自动带 Cookie。
4. 解析 HTML，去掉 script/style，优先提取 article/main，转 Markdown。
5. 默认截断到 30,000 字符，保留 pre/table，返回 `truncated`。
6. 记录 tool_call_id、url、final_url、status、sha256、bytes、耗时；不记录正文和 Cookie。
7. 同域限速 1 QPS，30 分钟缓存，支持 ETag。

返回结构固定为：

`{url, final_url, status_code, title, content_md, truncated, cached, fetched_at}`

## 踩坑点

- **DNS rebinding**：不能只“判断域名解析结果是否内网”，要用已校验的 IP 建立连接，避免客户端二次解析。
- **robots.txt 会卡住企业后台**：提供 `skip_robots` 参数，但 skip 时要单独写审计日志。
- **Readability 会删掉代码和表格**：保留 markdown 原文，并提供 `auto/article/html/text` 模式。
- **同域限速不能省**：Agent 可能连抓几十个同站 URL，限制 1 QPS，超时返回“稍后重试”。
- **不要继承系统代理和浏览器 Cookie**：工具进程要跑在独立环境变量里，禁止带上公司代理凭据。

## 可复用建议

- 先只做 GET + 静态正文提取，不要给点击、滚动、表单提交。
- 默认拒绝内网、IP 直连、非 80/443 端口；白名单用后缀匹配，如 `*.example.com`。
- 在 Agent 系统提示里写清：网页只能通过 `fetch_web_page` 获取，不许用其它网络途径。
- 采集服务放入独立容器，限制 egress 到目标域名段；审计日志接可观测平台。

## 总结

“稽客”不是爬虫，而是 Agent 对外部网页的安检口。价值不在读得多全，而在把风险压到可接受范围：默认拒绝、内容最小化、全程可审计。对 OpenClaw 这类自动化来说，薄薄一层边界比直接给模型万能钥匙更耐用。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/73b5eea0ee228731.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/a83b0dc2322121a9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/d8da35f652a233fe.png)

