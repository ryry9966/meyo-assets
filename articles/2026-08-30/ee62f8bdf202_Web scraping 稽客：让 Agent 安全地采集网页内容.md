---
title: Web scraping 稽客：让 Agent 安全地采集网页内容
feedId: 35434
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景

在 OpenClaw 这类 Agent 平台里，网页采集几乎是所有自动化任务的基础：查资料、读文档、抓发布页、监控变更。但很多实践里，“让 Agent 读网页”被简化成给模型一个万能 `http_get` 工具，甚至直接让模型生成脚本去跑。结果就是抓取行为不可控：内网探测、重复抓取、超大响应、编码乱码、反爬触发，最后变成排障黑洞。

工程上有必要把网页采集收口到一个“稽客”层。这里的“稽客”不是指某款爬虫产品，而是指一个负责稽核每次抓取请求的 sidecar/网关：每次 URL 进入前先做策略校验、请求中限速、响应后做内容清洗和审计。它类似 API 网关对爬虫下行流量的治理。

## 要解决的问题

1. **权限过大**：一个 MCP 工具可以访问任意 URL，包括 `localhost`、云元数据地址、内网 IP。
2. **没有限速与缓存**：同一 URL 被多轮 Agent 反复抓取，或对目标站点造成不必要的压力。
3. **HTML 噪声直接进入上下文**：脚本、样式、导航、广告混在一起，浪费 token，也干扰模型判断。
4. **失败不可观测**：超时、403、429、重定向、编码错误没有统一分类，重试策略粗暴。
5. **法律与合规边界模糊**：不尊重 robots、伪造 UA、无视缓存策略。

## 做法/步骤

我建议把抓取能力做成一个 MCP server 或 OpenClaw 插件，暴露一个工具，例如 `fetch_page(url, options)`。不要给开放式的 `exec` 或 `requests` 能力。

1. **请求前校验**：先解析 URL，做 DNS 解析，拿到实际 IP 后校验是否属于私网、回环、链路本地、组播等地址段。注意必须“先连接 IP、再校验 IP”，而不是只校验域名，否则 DNS rebinding 可以绕过。默认策略 `deny all`，按 host 或后缀显式加入白名单。配置文件示例：

```yaml
scraper:
  default_policy: deny
  allowed_hosts:
    - "*.example.com"
    - "docs.openclaw.cn"
  deny_private_ips: true
  max_bytes: 300_000
  timeout_s: 12
  max_redirects: 3
  respect_robots: true
  render_js: false
  host_rate:
    rps: 1
    burst: 3
```

2. **请求执行与状态机**：用 httpx 或等价库发起请求，统一处理 gzip、charset、状态码分类。对 403/404 记一类错误，对 429/503 记限流错误并做退避，不要盲目重试。关闭自动重定向，每一步重定向都重新做 host/IP 校验；跨域重定向默认拒绝或再次过白名单。

3. **内容管道**：响应成功后在服务端做语义压缩。去除 script/style/nav/footer/iframe，保留 article、main、pre、table 等结构，再转成 markdown。限制输出长度，超过阈值只保留前部并给出截断标记。返回一个结构化对象，包含 title、canonical_url、status、content_markdown、content_hash、fetched_at、warning_tags。

4. **审计日志**：每次抓取记录 trace_id、agent_id、host、url、status、耗时、字节数、缓存命中、风险标签。日志保留 30 天，便于回溯“模型为什么得出某个答案”。

5. **集成 OpenClaw**：在工具描述里明确边界，例如“只能访问白名单域名，不执行登录态请求，不渲染 JS”。系统提示里再加一句，要求 Agent 优先使用缓存或已抓取内容，避免重复请求。

## 踩坑点

- **DNS rebinding / SSRF**：如果只做“先解析再校验 IP”，攻击面仍在。连接阶段要使用校验过的 IP，并禁用或限制 80/443 以外的端口；对 IPv6 也要覆盖。
- **重定向跨域**：很多站点 http→https、www 规范、跳转到 CDN，默认拒绝会误伤。需要预置规范化规则，但仍要过白名单，不能信任重定向时的新 host。
- **编码与内容类型**：不要按 requests 的猜测定死，优先看 header charset，其次解析 meta。对 PDF、图片、压缩包等非 HTML 响应单独处理或直接拒绝，避免把二进制塞进模型。
- **动态页面成本**：大多数案例不需要渲染 JS。若遇到仅有 shell 的 SPA，再看是否开放 `render_js`，并设置 max JS 执行时间、关闭自动滚动、设置视口和等待条件。动态渲染也容易引入广告位和弹窗噪声。
- **反爬与 429/503**：稳定 UA 比频繁伪装更好；不要因为一次失败就提高并发。站点返回 429 时按 Retry-After 退避，连续失败就熔断该 host。
- **缓存副作用**：登录态内容、带 token 的 URL 不要进共享缓存；按 Accept-Encoding 和 UA 做变体缓存，避免移动端和桌面端混用。
- **提取误删**：代码块、表格、数学公式容易被通用抽取器误删。抽取后要核对内容指纹，若页面主要是代码示例，宁可保留 `<pre><code>` 结构。

## 可复用建议

- 默认 deny，显式 allow；私网 IP 检测放在 DNS 之后、连接之前，做成单独函数并测试覆盖。
- 给每个 host 建立令牌桶限速和短期内存缓存，缓存命中直接返回结构化结果，减少 Agent 等待。
- 转换后的 markdown 先做“语义截断”：保留开头和包含关键词的片段，而不是机械截断。
- 把 robots 解析和缓存做成独立模块，不要把 robots 视为安全边界，但把它作为合规信号。
- 对每次抓取加 trace_id，Agent 报错时可以携带，方便回溯。
- 把 JS 渲染作为 opt-in 能力，并要求显式声明域名和资源预算。

## 总结

Web scraping 稽客的本质，是把不可预测的 Agent 网页访问收口成一条可观测、可限速、可审计的管道。它不需要做成大而全的爬虫框架，而应该像一个 MCP 工具或 sidecar 插件，为每次 URL 请求做策略校验、内容清洗和风险标记。这样 Agent 能更安全地读网页，也不容易被自己的“好奇心”送进内网或反爬黑名单。对于 OpenClaw-CN 社区来说，这套思路更适合放到 MCP 插件里复用，而不是让每个 workflow 散落一堆抓取脚本。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/48d2a26a15b84a4e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/37ec260570379e2a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/fa3309633a66d133.png)

