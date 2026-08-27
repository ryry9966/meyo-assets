---
title: Web scraping 稽客：给 Agent 装一层安全抓取网关
feedId: 34942
source: 综合讨论
publishedAt: 2026-08-27
---

## 背景

在 OpenClaw 这类 Agent 工具链里，让模型读网页是一个高频需求：查文档、看 issue、抓公告、取页面摘要。但“能抓网页”和“安全地抓网页”是两件事。直接把 `requests.get`、`shell` 或某个无约束的浏览器工具交给 Agent，很容易把内网探测、登录态泄露、上下文爆炸和反爬风控一起带进来。

这里的“稽客”可以理解为稽核式抓取代理：Agent 只能调用一个或少数几个受控抓取工具，网络、协议、内容、体积、缓存都由工具层先检查，而不是由模型自由决定。

## 问题

实际接入时最常见的几个风险：

- **SSRF**：模型可能访问 `http://169.254.169.254/latest/meta-data/`、内网网关、本机端口。
- **登录态泄露**：如果抓取工具自带浏览器 profile 或 cookie，Agent 可能把个人账号页面内容抓进上下文。
- **HTML 噪声**：原始 HTML 会迅速占满上下文，大量脚本、样式、导航栏没有价值。
- **反爬与资源消耗**：无节制的重试、并发和动态渲染，既容易触发目标风控，也浪费本地资源。
- **输出不受控**：页面过大、编码异常、重定向到登录墙，都可能导致工具返回不可用结果。

## 做法：把抓取工具关进“稽客”层

建议不要在 Agent 配置里直接开放 `shell` 或通用 HTTP 工具，而是注册一个 MCP 工具或插件，例如 `web_fetch`，只接收 `url`，内部完成校验。

核心校验可以做成这样：

```python
ALLOWED_SCHEMES = {"http", "https"}
BLOCKED_NETS = [
    "127.0.0.0/8", "10.0.0.0/8", "172.16.0.0/12",
    "192.168.0.0/16", "169.254.0.0/16",
    "::1", "fc00::/7", "::ffff:127.0.0.1"
]

def checked_fetch(url: str) -> dict:
    u = urlparse(url)
    if u.scheme not in ALLOWED_SCHEMES:
        raise ValueError("scheme not allowed")
    if u.port not in (None, 80, 443):
        raise ValueError("port not allowed")

    # 解析后固定 IP，避免后续 DNS rebinding
    ip = socket.getaddrinfo(u.hostname, u.port or 443)[0][4][0]
    if is_private_ip(ip):
        raise ValueError("private address blocked")

    # 禁止自动跟随重定向；如需跟随，逐跳重新校验
    html = fetch_with_limit(url, timeout=5, max_bytes=1_000_000)
    text = extract_article(html)  # 去掉 script/style，提取正文
    return {"url": url, "markdown": text[:60_000]}
```

然后在 OpenClaw 的工具权限里做白名单收敛：

```yaml
tools:
  - name: web_fetch
    description: "Fetch a public web page and return extracted markdown."
    parameters:
      url: string
    allow:
      - "https://docs.openclaw.cn/*"
      - "https://github.com/*/issues/*"
```

具体字段按你使用的 MCP 插件规范调整，但思路是固定的：**Agent 只能看到工具，不能看到工具背后的网络能力。**

## 踩坑点

- **重定向绕过**：很多 HTTP 库默认跟随 302，`Location` 指向内网时会绕过第一轮校验。要禁用自动重定向，或在每一跳重新做协议、端口、DNS、IP 校验。
- **DNS rebinding**：域名第一次解析为公网，第二次解析为内网。简单做法是解析后用 IP 直接连接，并固定这个 IP，不要用域名做二次连接。
- **IPv6 和特殊地址**：只拦 IPv4 私有段不够，`::1`、`fc00::/7`、`::ffff:127.0.0.1` 也要拦住。
- **robots.txt 不是安全边界**：它只是礼仪和风控参考，不能拿来防 SSRF。
- **动态页面成本高**：headless browser 容易被反爬，且资源消耗大。能直接请求 API、sitemap、RSS 就不要渲染页面。
- **登录态页面**：不要让 Agent 接触 cookie 原文或共享浏览器 profile。需要登录的抓取，建议单独建隔离会话，抓完即弃。
- **上下文膨胀**：即使提取成 Markdown，也可能返回几万字。默认截断，必要时让 Agent 用搜索或分块读取，而不是一次全量塞入。

## 可复用建议

- 优先维护**域名白名单**；如果必须支持任意 URL，再叠加网络层 SSRF 防护。
- 非必要不开页面渲染，动态页面单独隔离，使用独立 IP 或代理池。
- 抓取结果缓存 5–15 分钟，避免 Agent 重复重试给目标站点造成压力。
- 所有抓取请求写审计日志：时间、URL、状态码、返回大小、是否命中白名单。
- 工具 schema 不要暴露 `header`、`cookie`、`proxy` 等参数，除非有明确业务需求。
- 返回 Markdown 或纯文本，不要返回原始 HTML；输出控制在 8k–12k token 以内。

## 总结

给 Agent 做 web scraping，本质上不是“能不能抓”，而是“把抓取关进一个可审计、可限制、可降噪的盒子里”。网络边界、输出体积、权限最小化这三件事做好，Agent 才能把网页当资料用，而不是变成内网探测器和爬虫。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-27/c874cb3629eeb381.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-27/4c61f74c322b0a84.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-27/6acb93872ccefa21.png)

