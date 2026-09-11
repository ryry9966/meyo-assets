---
title: Web Scraping 稽客：让 Agent 安全地采集网页内容
feedId: 37132
source: 综合讨论
publishedAt: 2026-09-12
---

## 背景

Agent 接上 MCP 的 fetch 或浏览器工具后，"能上网"很容易，"安全地上网"是另一回事。社区里反复出现三类事故：整页 HTML 塞进上下文，几轮对话就把 token 烧光；页面里藏着指令式文本，Agent 读到后执行了不该执行的动作；没有频控的高频抓取，把出口 IP 打进目标站黑名单。这篇帖子整理我们踩出来的做法：在 Agent 和网页之间加一层"稽客"——可配置、可审计的采集安检层。

## 问题拆开看

- **内容不可信**：网页是第三方写的输入，等同于不可信用户输入，可能携带间接提示注入。
- **体量不可控**：导航、广告、脚本、无限滚动页，原始 HTML 动辄几百 KB。
- **行为不可越界**：robots.txt、服务条款、频控是硬边界，不能交给模型临场判断。

## 做法：六步流水线

```
URL 解析 → 白名单/robots 检查 → 限速
→ 抓取(超时+体积上限) → 正文抽取 → 清洗与返回
```

1. **准入**：默认拒绝。生产环境配域名白名单，重定向的每一跳都重新过白名单；非 `text/html`、`application/json` 直接拒绝。
2. **限速**：按 host 做令牌桶（比如 0.5 QPS），429/503 指数退避并读取 Retry-After。
3. **抓取**：15 秒超时 + 512 KB 体积上限；带常规 UA，缓存 ETag/Last-Modified。
4. **抽取**：readability 类算法取正文，剥掉 script/style/iframe/HTML 注释，转 Markdown；超长页只留头尾。
5. **清洗**：转 Markdown 之后再扫一遍隐藏指令模式（"ignore previous instructions"类句式、不可见字符），命中即标记或截断。
6. **返回**：content + 固定结构的 metadata（status、chars、extractor、from_cache、truncated），让下游 Agent 能基于来源做置信判断。

配置示例：

```yaml
allowlist: ["docs.example.com", "arxiv.org"]
max_bytes: 524288
timeout_s: 15
qps_per_host: 0.5
render_js: false        # 仅对白名单内 JS 站点单独开启
truncate: { head: 6000, tail: 2000 }
```

## 踩坑点

- **注入不只藏在正文**：alt 属性、meta description、PDF 附文都可能是载体。清洗必须放在转 Markdown 之后做，因为转换本身就会重组文本。
- **编码**：GBK 老站点会 charset 检测失败，抽取器静默返回空串，要按响应头/meta 显式解码。
- **headless 浏览器是最后手段**：先屏蔽图片、字体、媒体请求，加载时间能差一个数量级；一旦开渲染，白名单要收得更紧。
- **缓存的两面性**：监控类任务拿昨天的缓存回答今天的问题，比慢更糟，按任务类型设 TTL。
- **robots.txt 是 host 粒度**：www 与非 www、各子域要分别检查，解析结果可缓存但别缓存过期。

## 可复用建议

- 抓取层保持"笨"：确定性代码做抽取、清洗、限速，LLM 只做总结，不让模型临场决定要不要忽略边界。
- 所有抓取落日志：URL、状态码、字节数、耗时、是否命中缓存，出事能复盘。
- 用站点快照做 fixture，抽取规则改动跑回归，防止"昨天还好好的"。
- 工具返回 schema 固定下来，多个 Agent/插件共用同一套稽客层，别各自为政。

## 总结

"稽客"的本质不是某个工具，而是一条原则：**把网页内容当作不可信输入处理**——和对待用户输入一样做校验、限界、留痕。Agent 负责理解与决策，安检层保证它拿到的每一字节都干净、够用、合规。先把这层做扎实，再谈更聪明的采集。欢迎在社区帖下贴出你们的抓取配置和翻车现场，一起补这份清单。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/6ae3c55b4b016f9d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/811137d95d6d716b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/e109a7dc708e420e.png)

