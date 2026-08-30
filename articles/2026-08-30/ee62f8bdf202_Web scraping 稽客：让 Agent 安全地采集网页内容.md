---
title: Web scraping 稽客：让 Agent 安全地采集网页内容
feedId: 35426
source: 综合讨论
publishedAt: 2026-08-30
---

# Web scraping 稽客：让 Agent 安全地采集网页内容

## 背景

在 OpenClaw/Agent 实践中，模型本身不能直接访问网络，通常依赖 MCP server 或插件提供 `fetch`、`browse` 类工具。如果直接给 Agent 一个裸的 `curl` 能力，它很快会变成 SSRF 入口、上下文污染源和稳定性黑洞。

“稽客”可以理解成一个受控的抓取中间层：Agent 只提交 URL 和约束，稽客负责校验、抓取、清洗，最后只回传适合进入上下文的文本。这个中间层不需要很复杂，但边界必须清晰。

## 问题

直接让 Agent 抓网页，常见有四类问题：

- **安全**：访问内网地址、云元数据、localhost、DNS rebinding。
- **资源**：响应体积不可控，HTML 可能数 MB，压缩炸弹会直接打爆内存。
- **内容**：导航、脚本、广告、Cookie 横幅混入上下文，浪费 token。
- **稳定**：超时、编码、重定向、反爬、非标准 MIME。

这些问题不适合靠提示词“让 Agent 自己注意”来解决，必须在工具层硬性拦截。

## 做法

以一个 MCP 工具为例：

```text
fetch_url(url, max_chars=12000, format="markdown")
```

实现步骤建议如下：

1. **校验 URL**：只允许 `http/https`，解析 scheme、host、port，拒绝 `userinfo` 和非标准端口。
2. **DNS 解析与 IP 过滤**：用 `getaddrinfo` 获取所有 IP，拒绝私有、回环、链路本地、保留地址。云环境额外封禁 `169.254.169.254`、`100.100.100.200`、`fd00::/8`。
3. **限定请求**：超时 10 秒，最大响应体 2MB，边读边限，不先 `ReadAll` 再限制。
4. **检查 Content-Type**：只处理 `text/html`、`application/xhtml+xml`、`text/plain`，其他返回拒绝。
5. **HTML 转 Markdown**：先用 readability 提取正文，再转 Markdown；移除 `script/style/noscript/svg/iframe`；删除 `data:` URI；截断到 `max_chars`。
6. **返回结构化结果**：`{ final_url, title, markdown, length, truncated }`，不要返回原始 HTML 和 headers。
7. **留审计日志**：记录域名、URL hash、状态码、字节数、耗时、调用方 session/tool，便于排障。

## 踩坑点

- **重定向要做多次校验**：限制最多 3 次，每次 30x 都要重新校验目标 IP。否则 `http://example.com` 跳转到 `http://169.254.169.254` 会绕过第一道校验。
- **DNS rebinding**：第一次解析是公网，第二次解析是内网。HTTP client 如果内部重新解析，你的 IP 过滤会失效。更稳的做法是自定义 `DialContext` 锁定已校验 IP，再连接该 IP，并正确设置 Host header 和 TLS SNI。
- **压缩炸弹**：`Content-Encoding: gzip` 解压后体积可能远超 2MB。必须边解压边计数，超限立即中断，不能解压完再判断。
- **JS 渲染页面**：很多页面没有 JS 不返回正文。无头浏览器成本高、风险大，建议默认只提供静态抓取；确需渲染时，把 `render_js` 做成独立可选工具，放进沙箱容器，禁止访问内网和写文件。
- **robots.txt 不是安全边界**：它可以作为频率参考，但不能依赖它阻止 SSRF。安全边界必须由 IP 过滤和 allowlist 承担。

## 可复用建议

- 对内部分析任务，优先使用域名 allowlist。deny 私有 IP 只是兜底，不是主要策略。
- 只返回 text/markdown，不要暴露 headers、原始 HTML、cookie 或重定向链。
- 相同 URL 加 ETag/hash 缓存，30 秒内直接返回缓存，减少重复抓取。
- 拒绝时返回明确的 `reason`：`blocked_private_ip`、`too_large`、`unsupported_content_type`，让 Agent 能调整策略，而不是盲目重试。
- 准备固定测试集：`http://127.0.0.1/`、`http://169.254.169.254/latest/meta-data/`、`http://[::1]/`、重定向到内网的公网 URL、超大 HTML 和压缩炸弹样本。每次改完校验逻辑跑一遍。

## 总结

安全 scraping 不是给 Agent 一个“浏览器”，而是给它一个边界清晰的文本获取器。把 URL 校验、IP 过滤、体积限制、HTML 清洗做成默认，再按需开放 JS 渲染。这样 Agent 可以稳定地读取公开网页，同时把 SSRF、token 浪费和反爬问题挡在上下文之外。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/b02991076859f4dd.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/ee5da5ff47c67a31.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/699d1e9831e01ff3.png)

