---
title: Web scraping 稽客：给 Agent 一个可控的网页采集工具
feedId: 35604
source: 综合讨论
publishedAt: 2026-09-01
---

## 背景：Agent 需要“手”去碰网页

在 OpenClaw、MCP 和插件式自动化场景里，Agent 最常见的诉求之一就是“帮我看看这个页面里有什么”。无论是做信息提取、价格监控，还是喂给下游模型做总结，都需要一个稳定的网页采集能力。

但如果直接把 `requests.get()` 或 `curl` 塞进工具函数，很快就会遇到一堆工程问题：被封 IP、抓了一堆 HTML 噪音、触发法律或 robots 边界、动态页面拿不到内容。Web scraping 不是一个“请求+解析”的单点动作，而是一套需要约束的策略。

## 问题：不安全的采集会反噬 Agent

一次失控的抓取至少会带来四类问题：

1. **合规风险**：无视 robots.txt、抓取受版权保护内容、高频访问造成对方服务压力。
2. **稳定性风险**：目标站点返回 403、重定向循环、超时、响应体过大，都可能让 Agent 卡住。
3. **内容质量风险**：HTML 里混着导航、广告、脚本，直接丢给模型会浪费 token 并引入噪声。
4. **可观测性风险**：没有日志、没有限流、没有审计，出了问题无法定位。

所以“稽客”的核心不是教你怎么绕过反爬，而是让采集行为变得**可预期、可限制、可追溯**。

## 做法：把采集拆成四层

我在实际项目里把 scraping 工具拆成四层，每一层都能独立替换和测试。

### 第一层：采集前检查

在发请求之前，先检查目标域名是否允许采集。解析 `robots.txt` 并缓存结果，避免每次重复请求。同时维护一个域名黑名单，比如登录页、支付页、个人信息页，Agent 无权访问。

```python
def check_allowed(url: str) -> bool:
    domain = get_domain(url)
    if domain in DENY_DOMAINS:
        return False
    robots = get_robots(domain)  # 带缓存和过期策略
    return robots.can_fetch(USER_AGENT, url)
```

这一层失败就直接返回一个结构化错误给 Agent，而不是让它自己猜。

### 第二层：请求控制

请求层要做五件事：

- 固定 User-Agent，并标识为自动化工具（不要伪装成浏览器）。
- 设置超时：连接超时 3 秒，读取超时 10 秒，总超时 15 秒。
- 限制重定向次数，最多 3 次。
- 限制响应体大小，超过 2MB 直接截断或报错。
- 每个域名做令牌桶限流，默认 1 req/s，可配置。

```python
limiter = TokenBucket(rate=1.0, capacity=2)
with limiter.acquire(domain):
    resp = session.get(url, timeout=(3, 10), allow_redirects=True, max_redirects=3)
    if len(resp.content) > MAX_BODY_SIZE:
        raise ScrapeError("response too large")
```

### 第三层：解析与清洗

拿到的 HTML 先去掉 `<script>`、`<style>`、`<nav>`、`<footer>` 等无关标签，再提取标题、正文、主要图片。推荐用 `trafilatura` 或 `readability-lxml`，它们对正文提取的效果比手工写正则稳定得多。如果页面是 JS 渲染的，再决定是否启动无头浏览器，但一定要加渲染超时和资源限制。

### 第四层：返回结构化结果

不要直接把原文塞给 Agent，而是返回一个 JSON：

```json
{
  "title": "...",
  "text": "...",
  "url": "...",
  "fetched_at": "...",
  "content_length": 1200
}
```

如果页面太大，截断到比如前 6000 字符，并告诉 Agent 原文被截断，让它可以决定是否继续请求。

## 踩坑点

- **robots.txt 解析不严谨**。很多库只处理 `User-agent: *`，遇到带 `Crawl-delay` 或多组规则会漏判。建议用 `robotparser` 并额外检测 `Crawl-delay`，如果站点有要求就降低限流频率。
- **限流状态在分布式环境失效**。工具跑在多个进程或节点上时，内存令牌桶不共享。需要把限流状态放进 Redis 或网关层统一控制。
- **动态页面成本失控**。无头浏览器渲染一个页面可能消耗几百 MB 内存和数秒时间。默认不要渲染，只有 Agent 明确说“页面内容为空”时再降级到渲染。
- **反爬不只是验证码**。很多站点会通过 TLS 指纹、请求头顺序、行为特征识别自动化流量。不要试图绕过，遇到 403 就返回错误给 Agent，让它换一个来源或告知用户。
- **版权和隐私边界**。采集来的内容不能直接用于再分发或模型训练，除非你有授权。在工具描述里写清楚“返回内容仅用于当前会话分析”。

## 可复用建议

- **用 MCP server 封装**。把上述逻辑做成一个 MCP 工具，Agent 通过标准协议调用，这样不同 Agent 框架（OpenClaw、Claude、自研）都能复用。
- **缓存页面快照**。对同一个 URL 在 1 小时内不重复抓取，既降低对方压力，也加快 Agent 响应。
- **支持意图声明**。让 Agent 在调用时传一个 `purpose` 参数，比如 `summarize`、`price_check`，工具可以据此调整截断长度或解析策略。
- **加审计日志**。记录每次抓取的 URL、时间、状态码、耗时、是否命中限流，便于事后排查。
- **准备 fallback**。如果目标站抓不到，可以返回该页面的元信息或让 Agent 尝试搜索引擎缓存，而不是死磕。

## 总结

让 Agent 安全地采集网页内容，本质上是把“自由抓取”变成“受控抓取”。它需要 robots 检查、限流、超时、内容清洗和审计日志的组合，而不是一个简单的 HTTP 请求函数。这套方案我已经在多个自动化工作流里稳定运行，核心就一句话：**给 Agent 足够的信息，但不要给它无限制的爪子。**

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/5708f5dc00c1b3ae.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/4a58975e951e1b53.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/85e9ad0d1bbd5e97.png)

