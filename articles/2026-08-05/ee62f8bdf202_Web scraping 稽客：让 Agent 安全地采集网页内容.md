---
title: Web scraping 稽客：让 Agent 安全地采集网页内容
feedId: 31728
source: 综合讨论
publishedAt: 2026-08-05
---

最近在给 OpenClaw 接入一个「检索增强」的 MCP 工具时，发现一个被反复提起但很少被认真对待的问题：**Agent 能不能安全地抓网页？**

OpenClaw 的 Agent 需要实时信息时，常见的做法是让它直接去访问 URL。但把一个能自由执行代码的 Agent 直接丢到互联网抓页面，等于让它穿着拖鞋去考场——它确实能跑起来，但随时会出事。

## 先看问题在哪

直接抓网页，会遇到四类典型问题：

1. **SSRF 风险**：Agent 很可能被诱导去请求 127.0.0.1、169.254.169.254 之类的内网地址。这不是幻想场景，公开网络上有大量专门投喂恶意链接的数据源。
2. **无超时**：Agent 抓一个不响应服务的地址，会一直挂在那里等，拖垮整条对话链路。
3. **Token 爆炸**：一个详情页的原始 HTML 可能有 500 KB，折算成 token 后一次就把上下文塞爆。
4. **内容污染**：导航栏、广告、脚本标签的噪声会干扰 Agent 对正文的判断，导致回答质量下降。

## 做一个「稽客」工具，而不是直接裸抓

我们的做法是给 OpenClaw 增加一个 Webscrape MCP 工具，核心设计很简单：**在外面放一道门卫，而不是把门拆了让人随便进出。**

工具结构大致是这样：

```ts
const server = new McpServer({ name: "web-稽客", version: "0.1.0" });

server.tool(
  "scrape_url",
  { url: z.string() },
  async ({ url }) => {
    // ① 只允许 http/https 协议
    const parsed = new URL(url);
    if (!['http:', 'https:'].includes(parsed.protocol)) return error('仅支持 http/https');

    // ② 屏蔽私有网络、链路本地地址
    if (isPrivateIp(await lookupIp(parsed.hostname))) return error('禁止访问内网地址');

    // ③ 带超时与重定向限制
    const resp = await fetch(parsed, { signal: AbortSignal.timeout(8000), redirect: 'manual' });

    // ④ HTML 净化：去标签、去导航、提取正文
    const text = await extractMainContent(resp);

    // ⑤ 截断：最多返回 4000 字符
    return { content: [{ type: 'text', text: truncate(text, 4000) }] };
  }
);
```

接入方式：在 `openclaw.app.json` 中声明 `mcpServers`，指向本地跑起来的这个 Node 服务即可。Agent 在需要查资料时就会自动调用它，而不是自己去乱抓。

## 实际踩过的坑

**坑 1：robots.txt 不判会封 IP**
刚开始没检查 robots.txt，连续爬了一个资讯站后 IP 被临时封了。后来加了缓存型的 robots 检查：只拦截明确 `Disallow` 的路径，不管 crawl-delay（Agent 场景不适合硬等）。

**坑 2：重定向是 SSRF 和 content 污染的暗门**
第一次校验了 `url`，但没处理重定向。攻击者可以给一个内网地址，服务端 302 跳进去。解决方法是 `redirect: 'manual'`，拿到 Location 后再次做同样的校验。

**坑 3：动态渲染的页面抓不到正文**
Vue/React 站点的 HTML 里只有 bundle.js，没有正文。这个坑没有银弹，只能加配置项：`force_scrape: true` 时，内部调用 headless browser。但要给这个能力配 `require_approval: true`，防止 Agent 无节制地调。

**坑 4：HTML 净化过度会丢结构**
一开始直接 `strip_tags`，结果把列表、表格全拍平了，Agent 读的时候丢语义。后来改为「白名单保留」：h1-h3、p、ul/ol、li、table、pre/code 保留，其余按块级/行内分派。

## 可复用的建议

- **策略放服务端，不放 prompt**：不要指望提示词约束 Agent。校验、超时、截断都必须发生在工具内部。
- **工具粒度要细**：`scrape_url` 只做一件事，不要学某些工具把搜索+抓取+提取塞进一个 tool。OpenClaw 的 Agent 工具选择依赖工具的语义清晰度。
- **缓存是必须的**：同一 URL 在会话中可能被反复读取，加一层 LRU（比如 50 条、TTL 30 分钟），既省钱又降低被封风险。
- **留日志**：每次抓取记 `url + 目标 IP + status_code`，这是排查问题时的保命符。

## 总结

让 Agent 安全抓网页，本质不是「加一个万能抓取器」，而是**用受控的工具夹住 Agent 的自由度**。校验入口、设定边界、截断输出——把这三个环节做好，就能在安全与实用之间取得平衡。

社区的插件生态里可能有现成的方案，但建议自己过一遍原理再接入，因为「安全」这件事，不能只依赖信任一个第三方包。

---

