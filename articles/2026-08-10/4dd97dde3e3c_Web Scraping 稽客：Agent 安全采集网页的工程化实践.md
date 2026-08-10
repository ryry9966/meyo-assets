---
title: Web Scraping 稽客：Agent 安全采集网页的工程化实践
feedId: 32347
source: 综合讨论
publishedAt: 2026-08-10
---

## 背景：Agent 采集网页的隐忧

在 OpenClaw 这类智能体平台上，几乎每个自动化场景最终都会碰到“帮我看看这个网页里的价格/新闻/表格”这类请求。于是我们为 Agent 接入了 Playwright、`curl`、`lxml` 等工具，让它能自由抓取互联网内容。功能很快跑通了，但几个工程问题随之暴露：

- Agent 可能重复抓取同一个域名，触发反爬甚至导致服务端 IP 被封；
- 采集动作未遵守 `robots.txt`，无意中踩中法律与合规红线；
- 返回的 HTML 中包含大量未脱敏的用户信息或追踪脚本，污染下游数据；
- 缺乏统一的速率控制与重试策略，错误处理散落在各工具脚本里。

这些问题无法靠“Agent 自己小心”来解决——智能体会忠实地执行 prompt，但 prompt 很难穷举尽所有安全边界。于是我们开始构建一个**稽客层**（Scrape Auditor），专门在所有采集调用前后执行安全、合规与质量检查。

## 问题拆解：稽客需要做哪些事？

稽客并非一个万能的反爬壳，而是一组轻量级策略执行点，明确切割职责：

1. **请求合规性审查**：目标 URL 是否在白名单或黑名单内？是否允许抓取该路径（对照 `robots.txt`）？是否需要遵守地域合规（如 GDPR 下的个人数据采集）？
2. **速率与并发控制**：同一域名两次抓取间最小延迟、最大并发请求数。
3. **内容清洗与脱敏**：移除 `<script>`、跟踪像素、PII（邮箱、电话号码等）裸露内容。
4. **可观测性**：每次采集记录决策日志（放行/拒绝/清洗原因），便于审计。

这些职责正好适合封装成一个稳定的中间件，对 Agent 暴露干净的工具接口。

## 落地步骤：从策略文件到可运转的审计工具

我们以 OpenClaw 插件体系为例，但思路同样适用于任意支持 function calling / MCP 的 Agent 运行时。

### 1. 定义审计策略

采用一份 YAML 配置文件描述站点规则，降低 Agent 对规则的感知成本：

```yaml
default:
  crawl_delay_ms: 2000
  max_concurrency: 2
  respects_robots_txt: true
  pii_mask: true

site_rules:
  - domain: "example.com"
    allowed_paths: ["/public"]
    disallowed_paths: ["/admin"]
    crawl_delay_ms: 5000
```

### 2. 构建稽客工具

基于 MCP 服务器实现一个 `scrape_audited` 工具，它内部执行以下流程：

```
Agent 意图 → 稽客决策引擎
               ├─ URL 过滤（白/黑名单）
               ├─ robots.txt 解析与验证
               ├─ 速率限制排队
               └─ 调用下游抓取器（Playwright/requests）
                     ↓
               内容后处理（清洗/脱敏）
                     ↓
               返回结构化结果（文本+元数据）
```

关键实现片段（简化）：

```python
async def handle_scrape(url: str) -> dict:
    domain = extract_domain(url)
    rule = load_rule(domain)

    if not is_allowed(url, rule):
        raise AuditDeny(f"{url} blocked by policy")

    await rate_limit(domain, rule.crawl_delay_ms, rule.max_concurrency)

    raw_content = await downstream_fetch(url)
    clean_content = sanitize(raw_content, rule)
    log_audit(domain, url, "allow")
    return {"content": clean_content, "source": url}
```

### 3. 挂载到 Agent 的工具列表

在 OpenClaw 中，只需将 `scrape_audited` 注册为函数工具，并在系统指令中说明“当需要网页内容时，使用 `scrape_audited` 获取”。Agent 就会自然遵从这一受限路径，无需逐一叮嘱。

## 踩坑点记录

实际部署时遇到了几个值得提前应对的问题：

- **robots.txt 缓存失效**：网站可能动态修改爬虫规则。我们采用了 TTL 为 10 分钟的本地缓存，配合解析库 `robotparser`（注意其不完美兼容部分格式，需要 fallback 到手动匹配）。
- **动态渲染页的稳定超时**：Playwright 等待 `networkidle` 常因无限轮询挂起。改为等待关键选择器 + 最大超时 15 秒，并允许 Agent 在失败后请求重试（但计速率）。
- **脱敏误伤**：粗暴删除所有 `@` 符号可能破坏合法的 Markdown 邮件引用。改用只脱敏疑似邮箱格式，并保留上下文提示“此处存在脱敏处理”。
- **并发饥饿**：当多个 Agent 实例同时抓取同一网站时，速率限制队列可能过长导致任务超时。引入令牌桶算法，并为不同 Agent 区分优先级（关键任务可手动提升）。
- **法律边界模糊**：某些公开数据（如公司注册信息）的爬取在部分地区属于灰色地带。稽客策略增加 `legal_note` 标记，在返回结果时附带提醒，由下游人类决策。

## 可复用的工程建议

1. **策略即代码，但配置应外部化**：把规则写成 YAML 或 JSON，使非开发者也能维护域名黑白名单，同时版本化管理，方便审计回溯。
2. **利用现有库缩短闭环**：`crawl4ai`、`scrapy-playwright` 等组合能快速搭起安全抓取基线，不要重写浏览器池。
3. **将稽客作为 MCP 服务独立部署**：这样无论 Agent 运行在本地还是云端，都能通过同一接口获得安全抓取能力，避免每个 Agent 实例重复维护策略。
4. **加入干运行模式**：Agent 请求采集时，可选只返回“若执行会抓取到的 URL 列表与合规状态”，用于预验证，减少无效调用。
5. **可观测性不是日志海**：记录每次审计决策摘要、拒绝原因统计，必要时通过 OpenClaw 的通知通道（如 webhook）告警异常高拒绝率。

## 总结

为 Agent 加上“稽客”这一层，本质上是把散落在 prompt、异常处理、运维脚本中的安全逻辑，收敛到一个可测试、可运维的工程组件里。它让网页采集从“黑盒调用”变成了可控、可审计的流程。工程团队从此不用再担心 Agent 半夜疯狂抓取 legacy 接口导致 IP 被封，合规负责人也能看到一份清晰的采集行为报告。

对 OpenClaw 使用者而言，这一模式完全可以扩展为可复用的 MCP 工具包，甚至构建一套社区标准：让智能体既拥有获取新鲜信息的能力，又不越雷池一步。

---

