---
title: Web Scraping 稽客：给 Agent 的网页采集装上安全护栏
feedId: 31683
source: 综合讨论
publishedAt: 2026-08-05
---

## 背景

Agent 跑起来之后，最频繁的挂载工具就是网页采集。调研竞品、抓取文档、收集舆情，全靠一次 `fetch_url`。但问题也随之而来：Agent 对链接没有“戒心”。

我在 OpenClaw 里跑过多步任务后发现，Agent 会顺着页面里的链接继续爬。一个看似正常的 `href`，可能指向登录墙、钓鱼站、追踪器链，甚至带恶意脚本的重定向。采集失控，不只是“爬到脏数据”的问题，而是整个 Agent 流程被不可信输入污染。

所谓的 **稽客**，不是攻击性的 hacker，而是给 Web Scraping 加上“稽核”这一层——让 Agent 的每次抓取都带审计日志、过滤规则和出站防护。这套思路对跑自动化任务的人非常关键：你不会希望自己的 Agent 在无意识状态下，替别人执行了 JS 挖矿脚本。

## 问题：Agent 采集的三重风险

1. **内容不可信**。抓回来的页面可能被篡改过。不是所有站点都像 MDN 那样干净。注入的 HTML 片段会骗过 Agent 的上下文理解。
2. **出站行为失控**。Agent 在 page 里发现新链接，跳转出去，访问了内网地址或 `.onion` 站点。日志里留下一堆不可控记录。
3. **反爬与账号风险**。高频抓取触发 WAF，导致 IP 被 ban。更麻烦的是，某些站点会把页面做成蜜罐，专门探测自动化访问者。

## 做法：让 Agent 的采集走三层过滤

我用的方案是 **OpenClaw 自带 HTTP 工具 + Playwright MCP + Falcon 扫描器**（或同类 URL 安全检测服务）组合。思路很简单：**采集前查信誉、采集中限出链、采集后过内容**。

### 步骤一：接入安全扫描工具

在 OpenClaw 里先把 Falcon（开源 URL 扫描工具）作为独立 MCP server 跑起来：

```json
{
  "mcpServers": {
    "falcon": {
      "command": "uvx",
      "args": ["falcon-mcp"],
      "env": { "FALCON_API_KEY": "your_key" }
    }
  }
}
```

启动后在 Agent 的 system prompt 里加一条规则：

> 在调用 `page_scan` 或 `fetch` 之前，必须先调用 `falcon.scan_url` 检查目标 URL。若返回 risk_score > 0.3，拒绝访问并说明原因。

这条规则不复杂，但能让 Agent 在“想抓就抓”之前多一步思考。实测下来，能挡住绝大多数钓鱼站和已知恶意域名。

### 步骤二：限制 Playwright MCP 的出站路径

如果你用 Playwright MCP 做 JS 渲染页面采集，默认情况下它允许任意跳转。需要改写 MCP 配置，把出站限制在目标域名白名单内：

```json
{
  "playwright": {
    "allowedDomains": ["docs.example.com", "api.example.org"],
    "blockedResourceTypes": ["image", "media", "font"],
    "navigationTimeout": 15000
  }
}
```

同时给 Agent 写死一个行为约束：**只抓取与主任务直接相关的页面。所有从页面内提取的新 URL，先返回给用户确认，不自行跟进**。这是最有效的一刀切策略——不是技术问题，是流程问题。

### 步骤三：内容抽取后做文本清洗

用 OpenClaw 内置的 `extract_text` 工具把 HTML 转成纯文本之后，记得再过一遍过滤规则。我的 filter 配置大概长这样：

```
INCLUDE: main article content
EXCLUDE: copyright, cookie notice, related links, injected scripts
```

如果是 Markdown 结果，建议检查有没有 `data:` URI 或 action 属性——这些是常见的注入歧途。

## 踩坑点

- **扫描器误报率不低**。Falcon 对部分海外站点域名解析慢，容易超时。建议给扫描加 2 秒超时和重试机制，不要因扫描失败阻塞主流程。
- **Playwright 的 UA 指纹太明显**。默认的 headless Chrome UA 一眼假，反爬强的站点会给假 HTML。需要手动设置真实的 UA 和 viewport。我踩过：某站点对 headless 返回 200 但正文全是一堆 JS 混淆代码。
- **MCP 工具返回内容过大**，超出 Agent 的 context window。建议抓完直接截断至前 8KB 文本，别让上下文被垃圾占满。
- 不要忽略 `robots.txt`——虽然它是君子协定，但采集频率失控时，对方直接封 IP 比什么都快。

## 可复用建议

1. **把“采集前校验 URL”作为一条默认 prompt 规则**，写进所有 Agent 项目的 base system 里，不要每个项目单独加。
2. **对采集结果做 Hash 记录**，存进日志。如果同一页面多次抓取结果 hash 不一致，说明页面在动态渲染或对爬虫有特殊对待，需要人工介入。
3. **受限域名清单**：在 OpenClaw 配置里维护一个 `blocked_domains.txt`，把已知广告联盟、挖矿脚本来源域和短链接服务都列入。
4. **采集频率做指数退避**，单域名请求间隔最低 5 秒。慢不会影响效率，被 ban 才是最大成本。

## 总结

Web Scraping 的下半场不是“抓得更快”，而是“探得安全”。给 Agent 装上稽核这层护栏，代价是一次额外的 API 调用、一条 prompt 规则和一点出站限制，换来的是数据可信度和行为稳定性的大幅提升。

做自动化实践的，别把 Agent 当成一个只会执行的哑巴工具。给它配备验证意识——一个会验证来源的 Agent，才配跑在核心业务链条上。

---

