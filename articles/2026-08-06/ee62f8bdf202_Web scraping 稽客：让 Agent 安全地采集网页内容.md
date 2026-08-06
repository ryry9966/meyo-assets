---
title: Web scraping 稽客：让 Agent 安全地采集网页内容
feedId: 31891
source: 综合讨论
publishedAt: 2026-08-06
---

## 背景：Agent 需要数据，但互联网不是免费自助餐

在 OpenClaw 的自动化链条里，网页数据是饲料。无论是构建知识库、竞品监控还是事实核查，Agent 总需要从真实页面中拉取结构或非结构化内容。原生的 `requests` + `BeautifulSoup` 组合能撑过原型阶段，可一旦把 Agent 放进生产环境，反爬、IP 封锁、合规红线就会扑面而来。

“安全采集”不是简单的换 UA 或睡几秒，而是**让 Agent 在规则内、可观测、可撤回地完成访问**。我们把这个负责把关采集行为的模块叫“稽客”——它像边境检查官，每次请求前后都要校验合法性，只放行合规操作。

## 问题：常规爬虫的 Agent 化困局

当 Agent 调用 MCP 插件去抓数据时，典型故障模式有三类：

1. **触发反爬决策链**：无头浏览器被检测到 `navigator.webdriver` 属性或 CDP runtime，直接被 WAF 垫回了验证码页，导致提取逻辑全空，浪费 token。
2. **频率过载连坐**：Agent 推理链路可能连续发起数十次请求，一个任务甚至会把整个 `/24` IP 段打进黑名单，关联业务跟着遭殃。
3. **合规盲区**：不检查 `robots.txt`，无视 `<meta name="robots" content="noindex">`，采集受版权保护内容或 PII（个人身份信息），给运营方埋法律雷。

这些都不是简单的代码问题，而是**自动化决策没有加入治理回路**。稽客的设计，就是把这个回路补上。

## 做法：构建稽客（Guardian Scraper）的三层架构

下图是我们的稽客在 OpenClaw 中的落位，它以 MCP 工具封装，Agent 不直接触及裸请求。

### 第 1 层：预检与规则引擎
每次采集指令下发前，稽客工具自动执行：

- 请求域名的 `robots.txt`，解析 `Allow`/`Disallow` 路径，匹配目标 URL。若 disallowed，直接抛出结构化错误（含规则引用），不发出任何 HTTP 请求。
- 检查本地缓存（Redis/文件）里的“站点规则卡”，包括：是否要求 JS 渲染、已知登录墙、限速建议（如 `Crawl-delay: 10`）。
- 将 URL 规范化为标准格式，杜绝 `https://example.com/../etc/passwd` 路径穿越。

```python
# 示例：稽客工具内部伪代码
path = urlparse(target_url).path
if not is_allowed_by_robots(domain, path):
    return {"error": "disallowed_by_robots", "rule": robots_line}
rate_limit = get_domain_policy(domain)["delay_seconds"]
await asyncio.sleep(rate_limit)
```

### 第 2 层：安全的采集通道
这一层负责落地请求，我们推荐 **Playwright 无头模式 + 专用 profile**，通过 MCP 服务器暴露给 Agent。关键配置：

- 启动参数：`--disable-blink-features=AutomationControlled` 并注入 stealth 脚本抹去 `navigator.webdriver`。但**不要过度伪装**——指纹一致性比隐藏更重要，否则更易触发高级检测。
- 网络节流：基于稽客返回的域策略，限制并发连接数（默认每个域 2 个连接），并启用请求间最小延迟（200ms~2s）。
- 会话隔离：每次 Agent 会话用独立 `context`，清除 cookies、localStorage，避免跨任务污染状态和身份关联。
- 响应验证：HTTP 状态≥400 自动留存快照，将异常 HTML 转存到 `mcp-logs/` 目录，方便回放排障而不是消失无痕。

### 第 3 层：采集后治理
数据到手，稽客还要做最后的质量与合规检查：

- **内容签名校验**：若 Body 包含已知的封禁文案（“403 Forbidden”“请开启 JavaScript”等），标记采集失败并触发二次重试（最多 1 次）。
- **PII 扫描**：基于正则和政策规则（中国居民身份证号、邮箱、手机号等）对文本进行检测，发现疑似 PII 自动脱敏或阻断存入长期记忆库。
- **使用声明注入**：将采集时间、来源 URL、robots 规则摘要作为元数据注入提取的结构化数据中，未来 Agent 引用时能追溯出处，满足合规审计。

## 踩坑点：从 demo 到生产掉的三个坑

### 坑 1：“robots.txt 说允许，但站点实际不欢迎”
某些站点 `robots.txt` 全 Allow，却在服务条款中禁止自动化采集。稽客在技术上拦不住法律风险。**解法**：Agent 配置文件里维护一份“受保护站点清单”，匹配域名直接拒绝抓取，人工审核后才能摘除。

### 坑 2：动态渲染的 Timing Hell
SPA 页面用 Playwright 打开后 `networkidle` 事件不可靠，数据未加载就提取。**解法**：稽客工具暴露 `wait_for_selector` 参数，由 Agent 推理出关键选择器传入，而非硬编码延迟。

### 坑 3：无头检测升级的军备竞赛
Cloudflare 等会检测 `navigator.plugins.length`、WebGL 渲染器指纹。**解法**：别追求全自动化隐身，改用 Playwright 连接真实 Chrome 并配合住宅代理，成本略高但稳定性远优于 stealth 插件依赖。

## 可复用建议

- **把稽客做成 MCP 插件**：参数仅为 `url`、`operation`（如 `fetch_html`, `extract_text`）、可选 `wait_selector`，Agent 别无其他权限。
- **接入现有代理池**：在插件配置中接入 `proxy_url`，并支持自动换 IP（根据 HTTP 429 响应触发）。
- **日志即审计**：每次抓取生成唯一 `trace_id`，将请求参数、robots 判定、耗时、最终状态写入 JSONL，便于事后分析 Agent 行为。
- **慢即是快**：严格遵循 `Crawl-delay`，并在全局维护并发令牌桶，防止 Agent 连锁触发高频请求。

## 总结

安全采集不是“攻防”，而是**让 Agent 的欲望服从互联网的节奏**。稽客模式把治理逻辑外挂为可插拔的 MCP 服务，让 OpenClaw 的自动化流程既有速度又有边界。工程上，三层预检-通道-后治理的架构，配合避坑经验，能显著降低生产事故。把这套方案沉淀成团队的标配插件后，Agent 的页面操作才能真正从“能用”走向“可信”。

---

