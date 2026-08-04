---
title: Web Scraping 稽客：让 Agent 安全地采集网页内容
feedId: 31631
source: 综合讨论
publishedAt: 2026-08-04
---

## 背景

OpenClaw 社区里不少 Agent 场景都依赖外部网页数据——竞品监控、舆情跟踪、资料汇总。但“给 Agent 一个网页 URL”和“让 Agent 拿到干净、可用的内容”之间，隔着反爬、登录墙、动态渲染、选择器失效这四座大山。更隐蔽的问题是：Agent 自主抓网页时会不会把自己暴露在风险里？UA 指纹、请求频率、Cookie 泄露，这些都是工程上必须回答的。

## 问题

先定义“安全采集”的三个层级：

1. **自身安全**：Agent 的出口 IP、UA、Cookie 不被目标网站记录或滥用。
2. **数据安全**：抓下来的内容被清洗、裁剪、去重，不把整站 HTML 直接塞进上下文。
3. **合规安全**：遵守 robots.txt，控制频率，不碰登录后内容。

很多时候我们只解决了“能抓到”，忽略了“抓了会不会出事”和“抓完能不能用”。

## 做法

我实现了一个 OpenClaw 插件（骨架代码 200 行不到），把“爬”和“取”解耦成两层：

**第一层：采集器（Scraper）**——用静态 fetch + 动态渲染双通道。

```python
# agent_scraper.py 核心逻辑
def safe_fetch(url):
    r = requests.get(
        url,
        headers={"User-Agent": "Mozilla/5.0 (compatible; OpenClawBot/1.0)"},
        timeout=10,
        allow_redirects=True,
    )
    # 限速：单域名 2 req/s
    time.sleep(0.5)
    return r.text
```

动态页面则走 Playwright，但统一包装成同一个接口，Agent 侧只调用 `fetch_web(url, mode="static|dynamic")`。

**第二层：稽客器（Validator）**——把 HTML 转成 Markdown，再过滤噪音。

```python
def validate_content(raw_html, domain):
    # 1. 用 trafilatura 提取正文
    text = trafilatura.extract(raw_html)
    # 2. 检查是否命中登录墙/验证码标记
    if any(k in text for k in ["captcha", "login required", "access denied"]):
        return {"status": "blocked", "content": None}
    # 3. 截断 + 去重
    return {"status": "ok", "content": text[:8000]}
```

这套设计在 OpenClaw 里的接入方式很简单：把插件函数暴露成 MCP tool，Agent 的 system prompt 里加一句 `采集网页时必须先调用 fetch_web 并用 validate 校验返回状态`。

## 踩坑点

- **选择器失效是常态**。CSS 选择器在目标站改版后必挂。解决方案是不要依赖选择器，改用正文提取算法（trafilatura / readability）替代。
- **动态渲染不是银弹**。Playwright 抓 SPA 确实有效，但并发跑 5 个实例很容易被 WAF 封 IP。建议加一个内存版 token bucket 做全局限流。
- **上下文被污染**。Agent 拿到整页 HTML 后 token 爆炸，且答案质量下降。必须让 validate 强制只返回正文摘要。
- **robots.txt 被忽略**。很多 Agent 框架压根不检查。在 fetch 之前做一次 `robots.is_allowed(url, user_agent)` 判断，成本极低但能规避大量法律风险。

## 可复用建议

1. 所有采集工具统一走同一出口，不要把 `requests.get` 散落在 Agent 代码里。
2. 每次采集结果落一份 JSON 缓存（key 为 URL hash + 日期），既是审计日志，也是可复用的数据资产。
3. 给 Agent 设定“抓取上限”：默认单次任务最多 20 个页面，防止 Agent 跑飞。
4. 对于需要登录的站，直接放弃。用公开数据做 80% 的场景，别为了 20% 的内容把自己暴露在封号风险里。
5. 错误处理要显式：`{"status": "blocked"}` 要告诉 Agent 换源，而不是重试三次。

## 总结

所谓“稽客”，就是让 Agent 的采集行为有稽可查、有规可循。安全不是靠绕反爬，而是靠约束 Agent 的行为边界。把采集拆成“抓取 + 校验”两层，再加上限速、缓存、robots 检查，这套方案足够应付 95% 的公开网页采集场景。代码量不大，但能把 Agent 从“裸奔爬虫”变成“守规矩的访客”。

---

