---
title: Web scraping 稽客：给 Agent 加一道网页采集安全闸
feedId: 34122
source: 综合讨论
publishedAt: 2026-08-22
---

## 背景

OpenClaw 这类 Agent 框架现在最常见的用法之一，就是让模型自己查网页、抓内容、做汇总。很多实践为了省事，直接给 Agent 一个 shell/http 工具，或者接一个“万能抓取”插件。结果是 Agent 什么都敢抓，内网地址、登录后页面、超大 HTML、反爬资源，一次任务里反复重试，最后把上下文打爆、IP 被封，甚至触发 SSRF。

我们在内部做了一个轻量的网页采集 MCP 工具，起名叫“scraping 稽客”。它不追求抓得多，只做一件事：所有网页采集请求必须经过一个安全闸口，统一限速、校验、清洗，再返回给 Agent。

## 问题

无约束网页采集在 Agent 场景下会放大几类问题：

- **SSRF 风险**：模型可能被诱导抓取 `http://169.254.169.254/latest/meta-data` 或内网管理页面。
- **合规与封禁**：不检查 `robots.txt`，高频抓取，触发目标站点风控。
- **上下文污染**：原始 HTML 带回大量 JS/CSS/注释/隐藏内容，影响后续推理。
- **资源不可控**：响应体积、超时、重试次数都没有边界。

这些问题不是“让模型小心点”能解决的，必须在工具层强制。

## 做法 / 步骤

我把“稽客”实现成一个 MCP 工具：`scrape_web`。如果你的 OpenClaw 实例已经接入 MCP 客户端，可以直接挂载。

### 1. 安全策略前置

所有 URL 只允许 `http/https`。真正发起请求前：

- 解析 DNS，拒绝 private / loopback / link-local / reserved IP，防止 SSRF。
- 限制重定向最多 3 次，且每次重定向后重新校验 IP，防止 DNS rebinding。
- 单域名最小请求间隔 1.5 秒，或并发数限制为 1。
- 响应体最大 1 MiB，超时 8 秒。
- 请求头带可识别 UA 和联系方式，不伪装成浏览器。

### 2. 工具核心逻辑

`scrape_web` 入参保持简单：

```json
{
  "url": "https://example.com/article",
  "selector": "main",
  "text_only": true
}
```

核心流程伪代码：

```text
check_scheme(url)
  -> resolve_dns(host, reject_private_ip=true)
  -> check_robots(url)
  -> fetch(url, timeout=8s, max_size=1MiB, max_redirects=3)
  -> recheck_final_ip()
  -> parse_html()
  -> clean_and_to_markdown()
  -> return structured_result
```

`robots.txt` 不允许时，返回结构化错误 `ROBOTS_DENIED`，而不是返回空文本。这样 Agent 能向用户解释原因，而不是反复重试。

### 3. 解析与清洗

优先用可读性算法抽取正文，例如基于 `article`、`main` 标签或文本密度。允许调用方传 `selector`，但选择器失败时自动回退到全文提取。

清洗阶段删除：

- `script`、`style`、`noscript`、`svg`、`iframe`
- HTML 注释
- `hidden` 元素
- 外链图片和多余导航

最终输出为 Markdown 文本，保留标题、段落、列表和链接。默认不携带 cookie 或 `set-cookie`。

### 4. 接入 Agent

在 MCP 工具描述里写清楚边界：

> 只抓公开页面；遵守 robots；不接受登录态；输出可能被截断；动态页面可能不完整。

这个描述会显著影响模型调用行为。模型看到明确边界后，会减少乱试和无效重试。

## 踩坑点

### SSRF 不只是拒绝内网 IP

要注意 IPv6、十进制/十六进制 IP、`0.0.0.0`、`localhost.` 等变体。更稳的做法是在 fetch 前和 fetch 后都校验最终解析 IP，防止 DNS rebinding。

### robots.txt 处理

`robots.txt` 本身也可能拉取失败。保守策略是：拉取失败时默认只允许根路径，或者直接拒绝，是否放行做成配置项。不要因为 robots 拉取失败就完全放开。

### 动态页面别一上来就无头浏览器

很多站点内容是 XHR/API 返回的。优先寻找接口，必要时才用 Playwright 等做渲染。必须渲染时，限制并发、关闭图片/字体/CSS 请求，并设置渲染超时。

### 429 / 403 处理

遇到限流后做指数退避，不要伪装成搜索引擎或频繁换代理强抓。给域名加冷却时间，避免后续任务继续触发风控。

### 选择器很脆

硬编码 class 的抓取随着站点改版很快失效。尽量返回全文，让 Agent 自己从 Markdown 里抽取所需信息。用户传 `selector` 失败时，回退到可读性提取。

### 编码和响应类型

响应头 `content-type` 不是 HTML 时，直接拒绝或只返回 metadata。gzip 解压要限制解压后的大小，防止压缩炸弹。非 UTF-8 内容转换时忽略无效字节。

## 可复用建议

- **收敛抓取入口**：移除或禁用 Agent 可用的通用 shell/http 工具，只保留 `scrape_web`。
- **维护域名策略表**：允许域、是否需要 JS、频率、冷却时间、robots 失败是否放行等，都做成配置。
- **增加审计日志**：时间、调用方、请求 URL、最终 URL、状态码、耗时、命中规则、返回字节数。保留 7–14 天。
- **写固定测试用例**：抓内网地址、重定向到内网、robots 禁止、超大响应、非 HTML、429 等，每次改策略回归。
- **给 Agent 明确失败语义**：区分 `ROBOTS_DENIED`、`SSRF_BLOCKED`、`TIMEOUT`、`TOO_LARGE`，让模型能向用户解释，而不是盲目重试。

## 总结

不要给 Agent 一个裸的 HTTP 客户端。“稽客”本质上是把网页采集变成受控、可审计、可降噪的 MCP 工具。安全与合规是采集的第一道工序，不是后续补救。对 Agent 来说，一个好的抓取工具不是“什么都能抓”，而是“抓得到就抓，抓不到就清楚说明为什么”。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/a9842edfdb2e429a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/b660710d09dfa797.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/288296e963f4358e.png)

