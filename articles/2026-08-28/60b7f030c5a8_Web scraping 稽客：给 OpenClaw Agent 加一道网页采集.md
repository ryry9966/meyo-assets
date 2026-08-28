---
title: Web scraping 稽客：给 OpenClaw Agent 加一道网页采集安全闸
feedId: 35073
source: 综合讨论
publishedAt: 2026-08-28
---

在 OpenClaw 里接网页采集，很容易把 Agent 当成“自由浏览器”：给它一个 URL 就抓，抓完直接往上下文里塞。小范围跑通之后，一旦换成真实任务，问题会集中暴露：请求打到内网地址、同域高频抓取触发风控、页面结构稍改就取不到字段、抓回来的 HTML 里全是脚本和 cookie 弹窗。我们需要一个中间层，把“能不能抓、怎么抓、抓完怎么给”收口。这个层可以叫“稽客”——它不是更强的爬虫，而是对每次抓取做审计、清洗和策略判断的守门人。

**问题拆解**

- 安全边界不清晰：Agent 可能请求 localhost、10.x、169.254.169.254、file:// 等地址。
- 抓取行为失控：没有频率控制、重试策略和单任务配额，连续请求同域触发 WAF。
- 内容污染：整页 HTML 直接进上下文，既浪费 token，又可能把网页中的指令当任务指令执行。
- 结构脆弱：写死 CSS 选择器，页面改版后静默返回空值，Agent 很难判断失败原因。
- 缺少证据：Agent 给了结论，但无法追溯到来源 URL、抓取时间和内容哈希。

**做法 / 步骤**

不要给 Agent 直接暴露 HTTP 客户端。建议在 MCP 或插件层只提供两个入口：

1. `fetch(url, mode, max_bytes)`：负责安全请求和页面清洗，返回 `{ok, final_url, status, content_type, charset, text/links}`。
2. `extract(url, schema, hints)`：负责结构化抽取，返回 `{ok, items, missing_fields, evidence}`。

`fetch` 内部流程如下：

- **URL 规范化**：只允许 http/https，禁止 userinfo、file://、gopher:// 等协议。
- **DNS 解析与 IP 校验**：解析所有 A/AAAA 记录，拒绝私网、回环、链路本地和云元数据地址；发生重定向后必须重新解析和校验。
- **请求控制**：固定独立 UA，单域并发不超过 2，请求间隔 800ms 起；遇到 429/403 返回可读错误，让 Agent 换源或停止，不自动硬刚。
- **页面清洗**：丢弃 script/style/noscript/iframe，移除隐藏节点，优先提取 main/article/body 的可读文本。超过阈值截断，而不是把剩余内容丢掉或直接报错。
- **无状态请求**：不保存登录态，不注入 Agent 侧的 cookie，不返回 Set-Cookie。

`extract` 的要点：

- 用 schema 描述目标字段。先走规则或选择器，失败后再用 LLM 从清洗后的文本抽取。
- LLM 只接收清洗后的文本，不接收原始 HTML 和页面脚本，降低提示注入风险。
- 每个字段附原文片段和 DOM path 作为 evidence。缺失字段超过 30% 时，返回“结构可能变更”，而不是空列表。

审计日志要落 JSONL，字段至少包括 domain、resolved_ips、final_url、status、elapsed_ms、bytes、content_hash、blocked_reason。这样后续排障、采样、评估数据质量都有依据。

**踩坑点**

- 只做 URL 字符串黑名单不可靠。短链接可以先跳公网再跳内网，IPv6 的 `::ffff:127.0.0.1`、十进制 IP 形式都可能绕过。必须在每次连接前基于解析后的 IP 做判断。
- 无头浏览器不等于更安全。它会把 localStoc、浏览器指纹和登录态带进去，还容易中页面反爬或提示注入。能用 HTTP 清洗流程就不上真实浏览器。
- 把 HTML 直接交给 Agent 是高风险操作。页面文本里可以出现“忽略系统指令”等内容，作为数据没问题，但混进提示容易出问题。
- 编码和压缩：只按响应头判断会乱码，不少中文站点在 meta 里才声明 GBK。工具要综合 header、meta 和 chardet 做解码。
- 不要把选择器写死且不做缺失检测。宁可返回“疑似改版”，也不要静默给空列表。

**可复用建议**

- 所有抓取走独立 egress 或代理出口，固定出口 IP，减少目标站风控和账号关联。
- 建立域名风险表：需要登录、验证码、法律声明或付费墙的站点，直接返回“需要人工确认”，不让 Agent 反复试。
- 给任务设总配额：每个任务最多抓 20 页，单域最多 5 页，发现循环抓取就断开。
- 返回内容永远带来源：URL、抓取时间、内容哈希、关键字段证据。这样 Agent 生成答案时可引用，排障时能定位。
- 先做只读模式：工具只输出“会如何处理该 URL”，不真正发起请求，便于验证规则是否拦截了内网和异常协议。

**总结**

稽客的核心不是替 Agent 爬更多网页，而是把网页采集变成可观测、可拒绝、可审计的动作。接好这个中间层后，Agent 仍能拿到所需内容，但不再拥有“任意访问 + 任意解析 + 任意入上下文”的自由度。对 OpenClaw-CN 的实践者来说，这一步比换更聪明的模型更能减少线上事故。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/d5442fc3c4320f61.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/b3f434ec0e3c5668.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/e79345d6a9b1f137.png)

