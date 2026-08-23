---
title: Web scraping 稽客：让 Agent 安全地采集网页内容
feedId: 34329
source: 综合讨论
publishedAt: 2026-08-23
---

# Web scraping 稽客：让 Agent 安全地采集网页内容

在 OpenClaw、MCP、插件和自动化脚本里，网页采集几乎是 Agent 的刚需。但直接让 Agent“打开网页、把内容抓回来”很容易失控：整页 HTML 塞进上下文、触发风控、忽略 robots、递归爬取、选择器三天两头失效。

这里说的“稽客”，不是让 Agent 变成爬虫，而是给它加一层受控采集层：Agent 只提意图，真正怎么抓、抓什么、抓多少，由工具层统一约束。

## 背景与问题

常见翻车点很集中：

- **上下文失控**：Agent 把整页 HTML 或导航菜单、广告脚本一起塞进 prompt，token 暴涨。
- **行为失控**：递归点链接、无频率控制、误触登出或表单提交。
- **合规风险**：忽略 robots.txt、Terms of Service，或采集到个人信息。
- **维护脆弱**：LLM 每次临时生成 CSS 选择器，页面小改就失效。
- **安全风险**：工具直接暴露 HTTP/浏览器能力，可能被诱导探测内网地址，产生 SSRF 类问题。

## 做法与步骤

在 OpenClaw 里，比较稳妥的做法是把采集能力封装成 MCP server 或插件工具，而不是给 Agent 一个裸的 `fetch` 或 `browse`。

1. **抽象工具签名**  
   例如只暴露：
   - `scrape(url, profile: "article" | "list" | "minimal")`
   - `can_scrape(url)`：检查域名是否允许、robots 是否放行、有无缓存。  
   Agent 不接触底层 HTTP 客户端，也不生成选择器。

2. **域名白名单与 SSRF 拦截**  
   工具层只允许配置好的域名，拒绝 IP、localhost、内网网段、短链接跳转后的非白名单域名。跳转要重校验。

3. **静态优先，动态按需**  
   很多文章、文档站用轻量 HTTP 客户端加 HTML 解析就够了，例如 Node 的 `cheerio` 或 Python 的 `BeautifulSoup`。只有明确需要 JS 渲染时才启动 Playwright/Puppeteer，并关闭图片、字体、CSS 请求，设置短超时。

4. **结构化返回，不返回原 HTML**  
   服务端完成正文提取或字段解析，返回 JSON/Markdown，例如：
   ```json
   {
     "title": "...",
     "published_at": "...",
     "main_text": "...",
     "links": []
   }
   ```
   不要让 LLM 从一堆 HTML 里自己找正文。

5. **限流、缓存与退避**  
   每个域名并发 1–2，最小间隔 1–3 秒；200 响应缓存 5–30 分钟；遇到 429/403 做指数退避。这样能避免同一 Agent 任务反复抓同一个 URL。

6. **请求身份与合规**  
   User-Agent 注明自动化用途和联系方式；缓存 robots.txt 并检查；登录墙、验证码、付费墙不要绕；采集结果落地前过滤邮箱、手机号等敏感字段。

7. **日志与预算**  
   记录每次采集的 URL、状态码、耗时、是否命中缓存、返回 token 长度。给单任务设置最大 URL 数、最大深度和总 token 上限，超出即停止。

## 踩坑点

- **裸 fetch 给 Agent 用**：它很快会尝试各种 URL、参数、编码，甚至探测内网。收口到工具内部是最有效的一步。
- **一上来就无头浏览器**：静态 HTML 能解决的场景没必要启动浏览器。Playwright 默认加载大量子资源，慢且容易被封。
- **复用用户浏览器 profile**：可能把登录 Cookie 带进采集请求，触发风控或越权。用独立 profile 更安全。
- **固定 sleep 等待动态内容**：用 `waitForSelector` 配合短超时，比写死 `sleep(3000)` 可靠。
- **忽略 robots 和频率**：测试环境不封不代表生产环境安全，尤其是小站点。
- **返回全文不做截断**：很多任务只做摘要或分块，不需要一次读完几万字。

## 可复用建议

- 工具层维护 schema 版本，页面改版只改解析，不改 Agent prompt。
- 将采集逻辑放在 MCP server 中，独立部署、可单测、可复用，比散落在插件脚本里好维护。
- 给 Agent 一个 `can_scrape` 前置检查，减少无效尝试。
- 为不同场景预设 profile：`article`、`list`、`minimal`，避免 Agent 每次发明新的解析需求。
- 对采集结果做敏感信息过滤和长度限制，默认只返回前 N 字符或摘要。

## 总结

Web scraping 稽客的本质，是让采集能力从“不可控动作”变成“可观测工具”。核心动作是：域名白名单、结构化返回、静态优先、限流缓存、日志预算。这样 Agent 能稳定获取网页信息，又不会把自动化项目变成一个随时可能被封、甚至越线的爬虫。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/ac72b1061451b700.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/448666908cad4d35.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/a76d3325fb707fe4.png)

