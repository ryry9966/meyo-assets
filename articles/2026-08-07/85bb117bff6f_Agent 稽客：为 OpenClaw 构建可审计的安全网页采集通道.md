---
title: Agent 稽客：为 OpenClaw 构建可审计的安全网页采集通道
feedId: 32002
source: 综合讨论
publishedAt: 2026-08-07
---

## 为什么需要给 Agent 戴上“稽客面具”

在 OpenClaw 的自动化编排里，我们常常需要让 Agent 自己从网页取数据：查文档、翻公告、比对价格、抓取更新日志。最直觉的做法是交给一个浏览器或 HTTP 插件，让它随便读。但这里的风险被严重低估了：

- **隐私泄露**：Agent 可能顺带把 Cookie、认证头暴露给第三方页面，尤其是 OpenClaw 运行在个人设备或内网时。
- **内容污染**：原始 HTML 里充满跟踪脚本、广告、隐藏的 SEO 文本，直接喂给 LLM 不仅消耗 token，还污染上下文，诱发幻觉。
- **滥用与合规**：没有审计的爬取会触发反爬、IP 封禁，甚至踩中内网探测（SSRF）的雷。
- **责任边界模糊**：当 Agent 访问了不该访问的页面，谁来承担后果？开发者需要在工具层划清界线。

所以我们需要一个“稽客”层——它不是反爬对抗工具，而是位于 Agent 和目标网站之间的安全审计中间件。它的职责是：**审核每次请求、净化返回内容、留下可追溯日志**，让采集变得可控、可解释。

## 问题拆解：一个单纯 fetch 的隐患

在 OpenClaw 插件生态中，直接用 `node-fetch` 或内置的 `web_fetch` 工具去拿页面是最常见的做法。但哪怕只是读一个“看似无害”的文档站，也可能：

1. 被 302 重定向到钓鱼页，Agent 照跟不误。
2. 返回的 HTML 中包含 1MB 的内联 SVG 广告，塞满上下文窗口。
3. 页面引用了 `file://` 或 `http://169.254.169.254/` 这类资源，触发了本地文件读取或云元数据泄露（若环境未完全隔离）。
4. 没有任何日志，事后完全不知道 Agent 读了什么、何时读、为什么读。

这些不是假设，是在真实自动化 pipeline 里经常踩到的坑。

## 设计思路：稽客作为请求的安全代理

我们的方案是把「访问网页」这个动作包装成一个受监督的 MCP 工具，我称之为 `scraper_auditor`。架构如下：

```
Agent → MCP Client → scraper_auditor (MCP Server)
                         │
                         ├─ 1. URL 审核（白名单 + 域名模式）
                         ├─ 2. 请求：无头浏览器 / HTTP（可配置）
                         ├─ 3. 内容净化（提取纯文本，剥离脚本/样式）
                         ├─ 4. 大小/超时截断
                         └─ 5. 审计日志（时间、Agent ID、URL、处理结果）
```

稽客工具对 Agent 暴露的接口非常简单：输入 URL 和可选说明（用于日志），输出净化后的纯文本或 Markdown。Agent 无需关心背后的清理逻辑，也无法绕过。

这样做的好处：**一次配置，所有 Agent 的网页访问都被收敛到同一个安全出口**，便于审计和限流。

## 可复现的实现步骤（以 OpenClaw 插件为例）

下面用 Playwright + Trafilatura 搭建一个最小可行的稽客 MCP 工具，可直接挂接到 OpenClaw 的插件目录下。

**1. 项目结构**
```
mcp-scraper-auditor/
├── package.json
├── tsconfig.json
└── src/
    ├── index.ts          # MCP Server 入口
    ├── auditor.ts        # 核心审核逻辑
    └── sanitizer.ts      # 内容净化
```

**2. 核心依赖**
- `@modelcontextprotocol/sdk` – MCP 服务端
- `playwright` – 无头浏览器（用于动态渲染）
- `trafilatura` – Python 库，通过子进程调用（或使用 Node 版 `@extractus/article-extractor` 简化）
- `p-limit` – 并发控制

**3. URL 审核与最终地址检查**
```ts
const ALLOWED_DOMAINS = ['docs.openclaw.cn', 'github.com', '*.wikipedia.org'];
const BLOCKED_PATTERNS = ['/admin', '/internal', 'localhost', '127.', '169.254.'];

function auditUrl(rawUrl: string): string | null {
  let url: URL;
  try { url = new URL(rawUrl); } catch { return null; }
  if (!['http:', 'https:'].includes(url.protocol)) return null;
  const hostname = url.hostname;
  const allowed = ALLOWED_DOMAINS.some(d => {
    if (d.startsWith('*.')) return hostname.endsWith(d.slice(1));
    return hostname === d;
  });
  if (!allowed) return null;
  if (BLOCKED_PATTERNS.some(p => url.pathname.includes(p))) return null;
  return url.href;
}
```
注意：实际打开页面后，必须再检查最终重定向后的 URL，防止白名单绕过。

**4. 无头浏览器请求与内容净化**
```ts
const browser = await chromium.launch({ headless: true });
const context = await browser.newContext({
  userAgent: 'OpenClaw-Auditor/1.0',
  bypassCSP: true,
});
const page = await context.newPage();

// 拦截不必要的资源
await page.route('**/*', (route) => {
  const type = route.request().resourceType();
  if (['stylesheet', 'font', 'image', 'media'].includes(type)) {
    return route.abort();
  }
  route.continue();
});

const response = await page.goto(finalUrl, {
  waitUntil: 'domcontentloaded',
  timeout: 15000,
});

const html = await page.content();
await context.close();

// 内容提取（这里用 trafilatura 命令行调用）
const text = await extractText(html); // 调用子进程或 Node 库
```

输出前，做长度截断（如 8000 字符）并移除零宽字符、过多空行。

**5. 审计日志**
每次调用记录结构化日志（JSON 行），包含：
- timestamp, agent_id, requested_url, final_url, status, extracted_length, duration_ms, error_message

这些日志后续可接入 OpenClaw 的可观测管道（如 Loki / Elasticsearch），用于回溯和告警。

**6. 注册为 MCP 工具**
```ts
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  if (request.params.name === 'scrape_audited') {
    const { url, reason } = request.params.arguments;
    const result = await performAuditedScrape(url, reason);
    return { content: [{ type: 'text', text: result }] };
  }
  throw new Error('Unknown tool');
});
```

再在 OpenClaw 的 `plugins.mjs` 里挂上这个 MCP Server，Agent 就能调用了。

## 踩坑记录

- **动态页面加载时机**：`domcontentloaded` 很多 SPA 页面内容不完整。设定 `networkidle` 又可能在持续轮询的页面上永远等不到。我们采用“domcontentloaded + 额外等待 2 秒 + 检查核心元素”的折衷策略，并设置硬超时 15 秒。
- **反无头检测**：部分页面（如 Cloudflare 防护站点）会检查 `navigator.webdriver`。稽客不是为了绕过反爬，遇到这种情况直接返回错误并记录，不该尝试注入反检测脚本，那会引入额外安全风险。
- **内存泄漏**：浏览器上下文用后必须关闭，否则长时间运行的 MCP Server 会把内存吃光。我们每处理完一个请求都关闭 context，并且限制并发数为 2。
- **内容净化不彻底**：`trafilatura` 能移除大部分模板噪声，但仍可能保留相对路径的链接。我们会做一层后处理，将所有 `href` / `src` 替换为占位符，避免 Agent 尝试“点进去”。
- **重定向放大**：原始 URL 是白名单，但重定向后可能跳转到黑名单。必须检查最终 URL。我们实现了一个最大重定向次数（5 次）和域名变化监控。

## 可复用建议清单

1. **永远不要让 Agent 接触原始 HTML**，只给净化后的文本或结构化摘要。
2. **白名单优于黑名单**：维护一份可访问的域名列表，并锁定协议（http/https）。彻底禁掉 `file://`、`data://` 等 scheme。
3. **拦截无关资源**：在浏览器层 block 图片、视频、字体、CSS，大幅减少带宽与渲染时间。
4. **为每次抓取设置配额**：单次内容上限 8000 字符、超时 15 秒、并发 2，并记录到日志。
5. **审计日志是救命稻草**：当 Agent 执行异常行为，“它到底读了什么页”是排障的第一线索。日志要结构完整、持久化。
6. **隔离运行环境**：将稽客 MCP Server 放在 Docker 容器中，与宿主机网络隔离，避免 SSRF 风险。
7. **将稽客视为“一次性”浏览器**：每次请求使用新的上下文，绝不重用 Cookie 或缓存，防止状态泄露。

## 总结

“稽客”这个概念的落点不是打造一个万能爬虫，而是为 Agent 的网页访问行为构建一个透明的安检卡口。对于 OpenClaw 实践者而言，它弥补了自动化链条中容易被忽略的安全与审计缺口，让“让 Agent 去网上读点儿东西”这件事从随意变得严谨。

当你把稽客插件挂上 MCP 后，你会感受到一种莫名的心安：Agent 仍然自由地从网上获取信息，但它走的每一步都有记录、有边界，不会在你不经意时翻开不该翻的页。

---

