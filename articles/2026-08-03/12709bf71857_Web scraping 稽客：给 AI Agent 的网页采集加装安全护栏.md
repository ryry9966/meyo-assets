---
title: Web scraping 稽客：给 AI Agent 的网页采集加装安全护栏
feedId: 31407
source: 综合讨论
publishedAt: 2026-08-03
---

# Web scraping 稽客：给 AI Agent 的网页采集加装安全护栏

## 背景

AI Agent 自动化实践中，网页抓取是最常见的“外部信息入口”。无论是让 Agent 读取新闻、检索技术文档，还是抓取实时数据，工程上都必须面对一个问题：**Agent 不懂得“规矩”，它只管发请求，不管目标网站受不受得了**。

直接让 Agent 通过 function call 去调用 `requests.get()` 或启动 `Playwright`，在原型阶段还过得去；一旦进入长时运行的自动化工作流，IP 被 Ban、触发 CAPTCHA、被 CDN 限流等问题会集中爆发。更危险的是，Agent 的调用逻辑可能无意间变成了 DDoS 工具——无上限的频率、重复请求、千篇一律的 UA，运维踩坑成本极高。

## 问题拆解：Agent 抓取为什么容易失控

把锅全甩给“目标网站反爬太严”是不合适的。更常见的原因是 Agent 端缺少一层**可控、可审计的采集中间层**。典型痛点包括：

- **频率失控**：Agent 短时间对同一域名发起数十次甚至上百次请求，完全不受限；
- **UA 固定**：全用 Python-urllib 或 `curl/7.68.0`，日志里一眼锁定；
- **忽略 robots.txt**：Agent never read robots.txt，可能触碰到法律与合规边界；
- **无缓存机制**：同一个 URL 被多个 Agent 或同一 Agent 的不同步骤反复拉取，加剧资源浪费；
- **反爬升级后无退路**：遇到 JS 挑战、5 秒盾，Agent 直接报错，整个自动化中断；
- **代理使用混乱**：把免费代理硬塞给 Agent，成功率极低，甚至引入恶意脚本。

如果不在 Agent 与目标站点之间设置“安全阀门”，抓取链路迟早成为运维地狱。

## 稽客中间层：思路与架构

“稽客”一词在这里指的是 **稽核 + 客制** —— 让每一次网页请求都经过审计与策略控制，同时为 Agent 提供简单、一致的调用接口。

实现方式上，我们选择了**独立部署的抓取代理服务**（Scrape Proxy），对外暴露为 MCP Server，OpenClaw 等 Agent 框架只需要调用这个 MCP 工具即可。架构如下：

- **Agent**（OpenClaw）  
  → 调用 MCP tool `scrape_proxy.fetch(url, options)`  
- **MCP Scrape Tool**  
  → 转发请求到 Scrape Proxy 服务的内网地址  
- **Scrape Proxy（核心）**  
  ├─ 白名单/黑名单检查 (domain allowlist)  
  ├─ 频率控制 (per-domain rate limiter，令牌桶)  
  ├─ 缓存查询 (Redis，URL 规范化后作为 key)  
  ├─ 代理池轮换 (从净化过的代理池中按权重取 IP)  
  ├─ UA 随机化 (预置 20+ 现代浏览器 UA)  
  ├─ 首轮尝试 HTTP 请求 (httpx)  
  ├─ 反爬检测/失败降级 → 启动 headless Playwright (有渲染引擎)  
  └─ 记录审计日志 (domain, status, latency, proxy used)  
- **Target Website**

所有策略由配置文件驱动，Agent 无需关心重试、代理、验证码等细节，只管拿最终 HTML。对于动态内容要求不高的场景，可以只保留 httpx 通道以降低资源消耗。

## 实操步骤要点

1. **定义策略配置**  
   为每个允许域名设置最小采集间隔、最大并发数、缓存 TTL、是否需要 JS 渲染等。例如：
   ```yaml
   domains:
     - host: "docs.example.com"
       rate_limit: 10/min
       cache_ttl: 300
       js_render: false
     - host: "spa.example.com"
       rate_limit: 5/min
       cache_ttl: 60
       js_render: true
   ```

2. **实现核心抓取与降级逻辑**  
   - 优先使用 `httpx` 发送普通请求，附加随机 UA 和代理。  
   - 如果返回状态码是 403、503，或检测到常见 CAPTCHA 标记，则切换到 Playwright 无头模式。Playwright 可以模拟真实浏览器指纹，但启动成本高，所以要限制并发数。  
   - 连接失败、超时自动重试，最多 2 次，重试时强制换代理。

3. **加入缓存与限流**  
   - 对所有抓取请求的 URL 进行规范化（去掉无关的 track 参数、统一大小写），存储 HTML 到 Redis。  
   - 使用令牌桶算法在应用层实现 per-domain 限流，避免因 Agent 逻辑 bug 导致突发流量。  
   - 限流触发时返回 429 给 Agent，让 Agent 按 Retry-After 自行退避（可在 MCP tool 中内置退避逻辑）。

4. **暴露为 MCP Server**  
   用 `mcp-python-sdk` 包装一个工具 `fetch_url`，输入参数包含 `url` 和可选的 `render_js`。内部直接调用 Scrape Proxy 的 `/api/v1/fetch` 接口。这样做的好处是 Agent 开发者不需要关心抓取服务的部署网络细节，只需要配置 MCP 连接。

5. **Agent 集成**  
   在 OpenClaw 的配置文件中，添加这个 MCP Server 作为数据源工具。Agent 在使用时就像调用普通 function tool，所有请求都会被稽客中间层约束。

## 踩坑清单

- **缓存命中但内容已过期**  
  对新闻类、实时类页面，缓存 TTL 需要分层。可以支持 `cache: false` 参数强制刷新，但仍受限流保护。
- **代理池“劣化”**  
  大量免费代理已经被重点风控系统标记，反而增加被封概率。建议采用支付服务的住宅代理，或者自建多个出口 IP 轮换。必须定期检测代理健康度，过滤掉高延迟、高封禁率节点。
- **robots.txt 解析不可靠**  
  不要自己写 robots.txt 解析器，使用标准库 `reppy` 但要注意它只遵循部分规则。最稳妥的做法是：一旦目标域名在 `disallow` 中明确禁止某些路径，就主动拒绝 Agent 的请求，并记入审计日志。
- **Playwright 降级成性能黑洞**  
  大量请求降级到 Playwright 会把容器内存吃爆。需要单独限制浏览器池大小，并对降级请求做二次限流。如果某个域名持续需要 JS 渲染，应该将其标记为“JS 优先”，直接走 Playwright 通道，减少无效的 httpx 尝试。
- **Agent 多次调用同一 URL**  
  虽然缓存能拦截大部分，但 Agent 可能附加不同的 query 参数或 fragment。最好在工具层做一次 URL 去重，避免“换汤不换药”的重复抓取。

## 可复用的工程建议

1. **独立部署，不要和 Agent 主进程混跑**  
   将 Scrape Proxy 作为独立 Docker 服务，方便扩缩容和监控。Agent 只通过 MCP 或 HTTP API 调用。
2. **监控与告警必不可少**  
   对每个域名的抓取成功率、平均延迟、降级比例、限流触发次数做 Grafana 面板。一旦某个域名成功率骤降，及时钉钉/邮件告警，提前切换代理或调整策略。
3. **策略即代码，纳入版本管理**  
   域名白名单、限流参数、代理配置等全部放在 Git 仓库，通过 CI 自动部署。修改策略走 PR 审核，防止口子越开越大。
4. **为 Agent 提供明确的错误语义**  
   返回给 Agent 的错误不应只是“fetch failed”，而应是结构化的：`{ "error": "rate_limited", "retry_after": 30 }` 或 `{ "error": "blocked", "suggestion": "try with js_render=true" }`，这样 Agent 可以做出合理决策（如等待、切换工具）。
5. **安全审计日志留底**  
   记录每一次抓取的目标域名、路径、时间、最终使用的 IP、耗时，方便后续合规审查和问题追溯。

## 总结

让 AI Agent 安全地采集网页内容，本质上不是反爬对抗技术，而是**工程化与控制面的问题**。一个薄而可控的稽客中间层，用最少的成本把无节制的自动化请求变成可监测、可约束的流量。对于长期运行、需要稳定获取 Web 数据的 OpenClaw 自动化实践者来说，这笔投入会在第一个被 Ban 的凌晨体现出它的全部价值。

---

