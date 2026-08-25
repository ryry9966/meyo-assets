---
title: Web scraping 稽客：让 Agent 安全地采集网页内容
feedId: 34674
source: 综合讨论
publishedAt: 2026-08-25
---

# Web scraping 稽客：让 Agent 安全地采集网页内容

## 背景

在 OpenClaw / Agent / MCP 自动化实践里，几乎不可避免要让模型读取网页：查文档、看 API 响应、抓列表、核对变更。最直接的做法是给 Agent 一个 shell 工具，让它 `curl` 或 `python requests.get`。这在原型阶段很快，但一旦 Agent 接触真实任务，就变成不可审计的网络出口。

“稽客”不是新的爬虫框架，而是一层很薄的采集审计中间件。它把“访问网页”从通用命令中剥离出来，做成只读、可限制、可记录的工具，让 Agent 只能通过它访问外部 HTTP(S)。

## 问题

直接给 Agent 原始 HTTP 能力，主要风险有五类：

1. **网络边界穿透**：访问 `localhost`、内网管理台、云 metadata 地址 `169.254.169.254`。
2. **资源失控**：超大页面、无限重定向、慢响应占满 worker。
3. **凭据泄露**：环境变量中的 cookie、Authorization 头被模型无意识带出。
4. **合规与反爬**：不控制频率、不遵守 robots/ToS，导致源站封禁或法律风险。
5. **数据质量**：返回整页 HTML 给模型，既费 token 又容易让模型抓错区域。

这些不是“以后加固”，而是 Agent 一旦能自主执行，风险就会立即出现。

## 做法

建议在 OpenClaw 中不要给模型 `shell` 或原始 `fetch` 工具，而是挂一个叫 `scraper` 的 MCP 工具。工具只暴露两个动作：

- `web_fetch(url, selector?)`：返回清洗后的文本/Markdown。
- `web_status(url)`：只做 HEAD/轻量 GET，返回状态和标题。

配置用一个 YAML 策略文件，由 gateway 读取：

```yaml
scraper:
  allow_hosts: ["docs.openclaw.cn", "example.com"]
  deny_private: true
  allow_ports: [80, 443]
  max_bytes: 1048576
  timeout_seconds: 8
  max_redirects: 3
  content_types: ["text/html", "application/xhtml+xml", "text/plain"]
  default_selector: "main, article, .content"
  cache_ttl_seconds: 300
  rate:
    per_host_per_minute: 20
```

关键实现顺序：

1. **先解析域名，再校验 IP，并用该校验后的 IP 建连**。不要先 `http.Get` 再查 `Response.Request.URL`，那已经晚了。
2. **对每次重定向重新执行 URL/IP 校验**。很多 SSRF 就是第一次 URL 合法，302 跳到内网。
3. **只允许 http/https，拒绝其他协议**，并在 dialer 层阻止私有/保留地址段。
4. **限制响应头/响应体大小，限制解压后大小**，避免压缩炸弹。
5. **默认剥离 cookie、Authorization 和自定义 header**。需要登录态时，用独立短期 token，且只在白名单域发送。
6. **返回前用选择器提取主内容，去掉 script/style/iframe**，转成 Markdown 或纯文本。
7. **写审计日志**：时间、Agent 标识、目标 URL、是否命中白名单、IP、状态码、响应大小、内容 SHA-256、耗时。不需要记录正文，避免隐私问题。

示意流程：

```text
Agent → MCP tool: web_fetch(URL)
          ↓
      allowlist 校验
          ↓
      DNS 解析 + IP 过滤
          ↓
      HTTP client（超时/大小/跳转限制）
          ↓
      HTML 清洗/选择器提取
          ↓
      纯文本/Markdown + 审计日志
```

## 踩坑点

实际落地中最容易忽略的是 **DNS rebinding**。攻击域可以第一次解析到公网 IP，第二次解析到内网 IP。如果客户端先验证域名解析 IP，再用域名重新发起连接，就会被绕过。所以要么用校验后的 IP 直接建连，并在 TLS SNI 保持正确；要么让解析器和连接器走同一缓存，确保“验证谁就连谁”。

另一个常见坑是 **IPv6 和地址表示变体**。`127.1`、`0`、`0x7f000001`、`2130706433` 都可能指向 `127.0.0.1`。如果只是字符串比较 `127.0.0.1`，很容易漏。建议统一 parse 成 net.IP 再判断 `IsPrivate()`、`IsLoopback()`、`IsLinkLocalUnicast()`、`IsUnspecified()`。

**重定向**也经常被忽视。很多实现只校验初始终点，不校验中间跳转。必须设置最大重定向次数，并对每一跳重新校验。

**登录态页面**不要直接让 Agent 用真实账号 cookie。建议为抓取账号开独立会话、最小权限、可随时吊销，且只允许访问需要登录的公开内容类型，不要把管理后台纳入。

**反爬**方面，即使域名合法，也要控制和源站的关系。UA 保持一致且可识别，不要伪装成浏览器；批量任务前至少检查 `robots.txt`；单域限速，失败指数退避。不要因为 Agent 很急就无限重试。

## 可复用建议

- 将“采集”做成最小权限 MCP 工具，而不是开放 `shell`。这是成本最低的安全边界。
- 配置放在版本库里，和 Agent 配置一起评审。白名单增加要走变更，避免模型说服你自己加域名。
- 优先返回提取后的文本，而不是 HTML。既减少 token，也降低把页面里潜伏的 prompt 注入带回上下文的风险。
- 每次请求都记录审计信息，保留 7–30 天。排查 Agent 行为时，这些日志比模型回忆可靠。
- 把私网拦截、大小限制和超时做成默认开启，不能通过 prompt 关闭。

## 总结

Web scraping 稽客的核心不是“爬得更快”，而是给 Agent 一个受控的采集出口。安全限制、资源限制、审计和结构化返回是四个缺一不可的部分。对 OpenClaw 实践来说，它可以是一个 MCP server、一个插件或一个内部 HTTP gateway；关键只有一点：不要给模型原始网络能力，让每一次网页访问都能被看见、被限制、被追溯。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/c6c4620e3c3783e0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/5bd71a55a366a03a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/af3001b7c954c189.png)

