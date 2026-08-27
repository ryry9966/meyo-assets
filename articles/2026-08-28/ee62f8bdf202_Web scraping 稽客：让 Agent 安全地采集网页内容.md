---
title: Web scraping 稽客：让 Agent 安全地采集网页内容
feedId: 34999
source: 综合讨论
publishedAt: 2026-08-28
---

# Web scraping 稽客：让 Agent 安全地采集网页内容

## 背景

在 OpenClaw 这类 Agent 工程里，让模型读取网页已经是常见需求。通过 MCP 工具或插件暴露一个 `scrape` / `fetch_url` 能力，模型就能总结文章、查文档、核对资料。问题在于：网页是外部不可信输入，而 Agent 拥有工具调用和推理能力，一旦把底层 HTTP 能力直接交给模型，风险会被放大。

常见的坑包括 SSRF、内网探测、云元数据读取、`file://` 读取、恶意重定向、慢响应、超大响应、页面内容注入等。更麻烦的是，模型可能被页面里的恶意文本诱导，或把“访问失败”理解成“换个姿势再试一次”。所以，Web scraping 需要一个独立守门层，而不是靠 prompt 反复叮嘱。

## 稽客的定位

稽客不是反爬工具，也不负责复杂页面解析。它是 Agent 与目标网页之间的一道执行边界：只解决“能不能访问、访问后能带多少内容回来、带回来内容是否安全”。

## 做法 / 步骤

### 1. 只暴露一个受控工具

参数只保留 `url`、`maxChars`、`timeout`，不接受自由传入 header、cookie、method。底层只允许 GET。不要给 Agent 一个完整 HTTP 客户端工具，否则模型可以组合出各种绕过路径。

### 2. URL 策略

- 仅允许 `http` 和 `https`，拒绝 `file://`、`gopher://`、`ftp://`、`data://`。
- 解析 URL 时拒绝 userinfo，例如 `http://user:pass@host/`。
- 如果 host 是 IP 字面量，直接进入私网校验；如果是域名，先做 DNS 解析，再对解析结果做私网校验。
- 必须拒绝的地址至少包括：`127.0.0.0/8`、`10.0.0.0/8`、`172.16.0.0/12`、`192.168.0.0/16`、`169.254.0.0/16`、`::1`、`fc00::/7`、`fe80::/10`。
- 优先走固定出口代理，让 DNS 解析和真实连接经过同一路径，减少 DNS rebinding 风险。

### 3. 跳转与请求策略

重定向最多允许 2 次，每次跳转后的新 URL 都必须重新执行 URL 策略。请求不要携带用户浏览器 Cookie 或 Authorization 头，UA 使用中性标识。设置连接超时 3 秒、总超时 8–10 秒，响应大小按解压后计算，上限建议为 1MB。

### 4. 响应与内容策略

先检查 `Content-Type`，只接受 `text/html`、`text/plain`、`application/json`、`text/xml`、`application/xml` 等文本型内容。流式读取并做截断，避免压缩炸弹或超大页面。抽取正文时丢弃 `script`、`style`、`iframe`、`object`、`meta refresh`。如果正文中出现大量私网 IP、云元数据路径或类似 AK/SK 形态的字符串，可以做脱敏或标记风险。

### 5. 返回结构

给 Agent 的返回使用 JSON，例如：

```json
{
  "ok": true,
  "finalUrl": "https://example.com/article",
  "status": 200,
  "title": "Example",
  "text": "正文内容...",
  "truncated": false,
  "notice": ""
}
```

错误不要回传原始堆栈，只给标准化错误码，如 `blocked_private_destination`、`too_large`、`unsupported_content_type`、`timeout`。

## 踩坑点

- **重定向绕过**：初始 URL 合法，但 302 到 `http://127.0.0.1:8080/`。必须在每次跳转后重新校验。
- **DNS rebinding**：第一次解析是公网 IP，第二次连接时变成私网 IP。真正的连接前应固定解析结果，或使用带 rebinding 防护的出口代理。
- **非标准 IP 写法**：`http://2130706433/`、`http://0x7f000001/`、`http://[::1]/` 可能绕过简单字符串过滤。应使用标准 URL/IP 解析器处理。
- **云元数据地址**：`169.254.169.254` 必须默认拒绝。
- **慢响应**：服务器一字节一字节地返回，超时后要断开，不要一直等。
- **无头浏览器插件**：如果使用 headless browser，只加载主文档，不要下载图片、CSS、字体、附件，否则容易造成资源消耗或文件下载问题。
- **模型重试**：工具描述里写清楚边界，不要让模型把策略拒绝理解成“网络坏了”后不断重试。

## 可复用建议

把稽客做成独立 MCP server 或 OpenClaw 工具包装器，策略配置化，不要散落在 agent 脚本里。默认拒绝，只有明确配置允许时才放行。至少包含这些配置项：协议白名单、私网段、最大跳转次数、最大字节数、超时、Content-Type 白名单。

保留审计日志：原始 URL、最终 URL、状态码、耗时、大小、拒绝原因。排障时比模型输出可靠得多。

测试用例固化成小脚本：`http://127.0.0.1:3000` 应拒绝；`http://169.254.169.254/latest/meta-data/` 应拒绝；合法公网页面应正常返回。如果目标站点需要登录态，不要把自己的主账号 Cookie 塞进 scrap 工具，使用单独的低权限账号或公开页面代理。

## 总结

Web scraping 稽客的核心不是限制 Agent 的能力，而是把网络访问变成可预测、可审计、有边界的工具。对 OpenClaw 或 MCP 插件来说，这一层应该放在工具入口和出口处，而不是只靠 prompt 约束。工程上先做到“不乱采”，再谈“采得好”。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/76afe24d8e668393.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/6cec670f4ed69849.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/587ba0f3288e1ffb.png)

