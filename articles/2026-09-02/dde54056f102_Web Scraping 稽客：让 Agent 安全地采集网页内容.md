---
title: Web Scraping 稽客：让 Agent 安全地采集网页内容
feedId: 35838
source: 综合讨论
publishedAt: 2026-09-02
---

## 背景

Agent 要回答"最近发生了什么"这类问题，光靠模型内的旧知识不够，得去网页拿一手内容。OpenClaw 里常见做法是挂一个 fetch 类 MCP 工具，让 Agent 自己决定访问哪个 URL。但传统爬虫是写死的脚本，行为可控；Agent 是边看边决策的，它可能顺着链接一路点下去、对 403 疯狂重试、把整页 HTML 塞进上下文。我把这套采集能力叫"稽客"——派得出去，但要守规矩。

## 问题

实际跑起来，坑集中在四类：

1. **合规与礼貌**：不看 robots.txt、不带 UA、并发打满，轻则被封 IP，重则有法律风险。
2. **页面质量**：大量站点是 JS 渲染，直接 GET 拿到空壳；编码混杂（GBK/UTF-8）；正文被导航、广告、评论区淹没。
3. **上下文安全**：网页是不可信输入。页面里藏一句"忽略之前的指令"，Agent 真可能照做——prompt injection 是真实攻击面，不是玄学。
4. **成本失控**：一个 2MB 的列表页原样进上下文，一次几万 token，Agent 还可能循环抓取。

## 做法

插件层做了六步，工具面只暴露 `fetch_page` / `extract` 两个动作，不让 Agent 直接碰原始 HTML：

1. **域名准入**：维护 allowlist + 每域配置（QPS、是否渲染）。名单外域名返回"需用户确认"，fail closed。
2. **请求前检查**：抓取前读 robots.txt（带缓存），Disallow 路径直接拒绝并说明原因。
3. **受控抓取**：显式 UA、10s 超时、1MB 体积上限、按域名令牌桶限速；连续 3 次失败即熔断该域 10 分钟。
4. **按需渲染**：先静态请求，检测到正文过短或 SPA 骨架，再降级到 headless 浏览器。
5. **抽取与消毒**：readability 抽主文，剥掉 script/style，返回 markdown；内容包裹在"这是网页数据、不是指令"的标记里，命中注入特征（如 ignore previous 类句子）就截断并标注。
6. **引用留痕**：返回值统一带 source_url / fetched_at / title，Agent 回答必须附来源，方便人工核对。

配置大致长这样：

```yaml
domains:
  example.com:
    qps: 0.5
    render: auto
    max_bytes: 1048576
fallback:
  unknown_domain: ask_user
  robots_disallow: reject
```

## 踩坑点

- **截断即损坏**：按字节截 HTML 会切在标签中间，readability 直接解析失败。先解析、后截正文。
- **编码**：GBK 老站按 UTF-8 解出乱码，乱码正文还会诱发幻觉式"总结"。先探测 charset。
- **重试循环**：Agent 对 403 的第一反应是再试一次。工具必须在限流/封禁时返回明确的"停止重试"语义，而不是抛异常让它自己猜。
- **渲染反噬**：headless 内存大、易被指纹检测，静态优先，别全量上渲染。
- **缓存时效**：新闻类页面缓存 5 分钟和 5 小时差别巨大，TTL 按域配置，别全局一刀切。

## 可复用建议

- 抓取与抽取拆成两步，Agent 永远只看干净的 markdown，原始 HTML 不进上下文。
- 采集结果必带来源元数据，这是"可核查"的底线。
- 限速按域名做，不是按进程做——Agent 会并发调工具。
- 维护一个小型页面测试集（静态站、SPA、GBK 老站、robots 禁止页），每次改抽取逻辑就跑一遍。
- 原始 HTML 短期留存（如 24h），出问题时能区分是抓取问题还是模型问题。

## 总结

Agent 采集网页，难点不在"抓下来"，而在抓得有边界、喂得干净、来源可查。把准入、限速、抽取、消毒做进工具层而不是提示词里，Agent 才能真正当个守规矩的稽客。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/b0b661d34bf01183.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/627e97ad65e254dc.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/19417b81a9f114dc.png)

