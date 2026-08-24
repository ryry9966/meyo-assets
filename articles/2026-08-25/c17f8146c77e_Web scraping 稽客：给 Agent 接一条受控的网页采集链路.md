---
title: Web scraping 稽客：给 Agent 接一条受控的网页采集链路
feedId: 34624
source: 综合讨论
publishedAt: 2026-08-25
---

## 背景

在 OpenClaw 这类 Agent 工程里，很多自动化场景需要“看网页”：读文档、抓商品信息、提取公告、做竞品巡检。最危险的实现不是抓不到，而是让 Agent 直接调用 shell 里的 curl/requests，甚至让模型自己拼 URL 和解析 HTML。这种链路会把网络权限、Cookie、内网地址、原始 HTML 全部暴露给推理循环，一旦遇到 prompt 注入或 URL 诱导，很容易变成 SSRF、数据泄露或上下文爆炸。

所以我会把网页采集从 Agent 主循环里剥离，做成一个独立工具“稽客”（scrape gatekeeper），只通过 MCP tool 或插件暴露有限能力。Agent 只描述“要哪个页面、要什么格式”，不直接碰网络。

## 问题

实际落地时，不安全采集通常表现为几类：

- 上下文爆炸：直接把整个 HTML 塞给模型，script/style/导航栏占掉大量 token。
- 出网边界失控：允许 file://、gopher://，或能访问 127.0.0.1、169.254.169.254、内网管理面。
- 频率和合规不可见：不读 robots.txt，多个 Agent 并发抓同一站点，触发 429 或封禁。
- 重定向绕过：首跳 URL 合法，30x 后跳转内网，DNS 只校验一次。
- 渲染过重：所有页面都上 headless browser，导致任务超时、资源耗尽。

“稽客”要解决的不是“能不能抓”，而是“抓得有没有边界、能不能审计、会不会把 Agent 带崩”。

## 做法

### 1. 只暴露一个工具接口

不要给 Agent 裸网。用 MCP tool 暴露：

```json
fetch_page(url, format, max_bytes, timeout, follow_redirects)
```

或面向结构化任务：

```json
scrape_structured(url, schema)
```

工具描述要写清楚：只允许 http/https，不访问内网，默认遵守 robots、单页上限、超时默认 10s。Agent 通过工具返回的 Markdown/JSON 决策，而不是自己解析 HTML。

### 2. 先做网络边界和 SSRF 防护

在采集器内：

- 只放行 http/https。
- 对域名做 DNS 解析，解析后校验每个 IP 是否属于 private/loopback/link-local/metadata 段。
- 每次 redirect 后重新解析、重新校验 IP，防 DNS rebinding 和 30x 绕过。
- 采集器放在独立容器/worker，使用单独 egress IP，不允许访问宿主机元数据。

注意：robots.txt 不是安全边界，不能作为 SSRF 防护；它只是礼仪和合规依据，安全判断要独立做。

### 3. 做请求治理

默认带规范 UA，解析 robots.txt 的 allow/disallow 和 crawl-delay；对 429/503 做指数退避；限制站点并发和总频次。高频站点加缓存 TTL，避免多个 Agent 重复抓同一个 URL。

### 4. 内容提取与清洗

优先使用轻量 HTTP + parser；只有明确是 SPA 或需要 JS 渲染时才启用 headless browser，但要在浏览器层拦截图片、字体、媒体、外链分析脚本，限制执行时间和页面大小。清洗时去掉 script/style/iframe/object/embed，输出 Markdown 或纯文本。原始 HTML 可落对象存储，但不进模型上下文。若页面超大，按 max_bytes 截断并在返回里标记 `truncated: true`，防止模型误以为内容完整。

### 5. 可观测与审计

每次采集记录 URL、状态码、耗时、字节数、是否命中 robots、重试次数、最终 IP、输出 hash。日志里绝不写 Cookie、Authorization、Set-Cookie。保留原始响应用于排障，但对敏感 header 做脱敏。

## 踩坑点

- 不要只做域名黑名单。内网 IP、云元数据、短链跳转、DNS rebinding 都会绕过。
- 重定向是最容易漏的一环。必须每个 hop 重新校验 IP。
- headless browser 要默认不启用，否则一个页面带几十个第三方脚本就会拖垮 worker。
- 忽略 `truncated` 标记会让模型产生“页面只有这些内容”的错觉，导致错误分页或漏数据。
- 采集器不要与 Agent 同一个特权网络。即使 URL 校验失误，也不应能访问内网。
- 登录态不要交给模型。用独立会话池或 OAuth 授权，避免账号密码和 Cookie 进入推理链路。

## 可复用建议

- 输出格式优先 Markdown/文本；只有结构化任务才用 schema 抽 JSON。
- 对同一站点配置缓存和并发上限，Agent 侧只关心“拿没拿到”，不关心“怎么拿”。
- 对 robots disallow 的 URL 默认拒绝；如果业务必须抓，走人工审批加入例外，而不是让 Agent 绕过。
- 采集器返回错误时要语义化：`blocked_by_robots`、`timeout`、`too_large`、`redirect_blocked`，方便 Agent 做下一步决策。
- 设置单页上限 256KB 已经够多数文档和商品页；高质量正文通常在 10-50KB。

## 总结

Web scraping 稽客的核心不是“更强的抓取”，而是把抓取变成受控能力：Agent 只拿到一个可校验、可限流、可清洗、可审计的工具，而不是裸网络权限。工程上先守住 SSRF 和出网边界，再做渲染与提取，最后通过日志和限制保证可回放、可排障。这样 Agent 去“看网页”时，才不会把自己和环境一起暴露出去。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/0c33848289b1d36d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/c4d6acc68ebc2d36.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/a6a09829815d5629.png)

