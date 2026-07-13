---
title: 基于 OpenClaw 的微博超话同步实践：图片上传、限流与状态归档
feedId: 28914
source: 综合讨论
publishedAt: 2026-07-13
---

## 背景与目标

在内容分发场景中，需要将生产端（例如资讯采集、AI 生成）产出的图文自动同步到指定微博超话。手工发帖不仅效率低，还容易因限流导致发布失败或重复发布。我们基于 OpenClaw 的 Agent + MCP 体系构建了一条同步管线，实现本地内容到超话的自动推送，重点解决三个技术问题：

- 图片资源的高可用上传
- 微博开放接口的频率限制与异常恢复
- 同步状态的可追溯归档，避免重复发布

本文不涉及具体业务内容，只谈管道与工程化踩坑，供正在用 OpenClaw 做社交平台自动化的同学参考。

## 问题拆解

完成一次超话同步，实际上要串起多个动作：获取内容 -> 上传图片 -> 组装帖子 -> 调用发帖接口 -> 记录同步结果。每个环节都可能因为网络、鉴权、限流等因素中断，需要补偿机制。

图片上传是最容易被低估的环节。微博的图片上传接口并非标准的 OAuth2 流程，即使使用官方 API，许多历史文档已失效，且对请求头有隐式要求。如果直接复用通用 HTTP 客户端，十有八九会收到 `302` 重定向或 `403`。

限流方面，微博对单用户短时间内的发帖、上传操作都有严格限制，黑盒策略明显，需要根据实际响应码动态调整节奏。

状态归档则是长期运行的自动化任务必需的：什么帖子发过、哪一步失败、重试次数、最终状态都需要落地，否则要么漏发，要么同一内容发多次触发风控。

## 做法与步骤

整个实践围绕 OpenClaw 的 MCP 服务器展开，把微博操作封装为工具，供 Agent 调用。

### 1. 微博 MCP 工具设计

在 OpenClaw 生态中，通过自定义 MCP Server 暴露三个工具：

- `upload_image`：接收本地图片路径，返回图片 ID（pid）
- `publish_status`：接收文本、超话 ID、图片 pid 列表，发布微博并返回帖子 mid
- `check_quota`：查询当前用户剩余发帖频率（解析响应头或特定接口）

核心鉴权采用 Cookie + CSRF 方式，从浏览器中导出登录态后注入请求头，避免频繁模拟登录触发验证码。

### 2. 图片上传的可靠实现

接口地址：`https://picupload.weibo.com/interface/pic_upload.php`  
关键要点：

- 必须使用 `multipart/form-data`，字段包括 `pic1`（文件）、`app`（`miniblog`）、`mime`、`data`（`1`）、`url`（必须传入对象字段，无则留空 `0`）、`markpos`（`1`）
- 请求头必须携带 `Referer: https://weibo.com/`，`X-Requested-With: XMLHttpRequest`
- Cookie 中需要 `SUB` 和 `SUBP` 等关键字段，同时带上从微博首页获取的 `X-CSRF-TOKEN`（可从 Cookie 的 `XSRF-TOKEN` 中取得，也可以从页面正则匹配）

成功响应格式为：

```html
<script>parent.url({"code":"A00006","data":{"pics":{"pic_1":{"pid":"xxxxxx","size":"...","width":...,"height":...}}},...});</script>
```

需要用正则提取 `pid`。这部分不能按 JSON 解析，很多客户端在这里踩坑。

在 MCP 工具中，我们将图片二进制流用 `requests` 或 `httpx` 构造 multipart 消息，设置好所有 header，并捕获异常进行三次重试。

### 3. 发帖接口与超话关联

发帖使用 `https://weibo.com/ajax/statuses/update`（Ajax 风格接口），POST 体包含：

- `text`：帖子内容（需要 URL encode，内容最后可拼接 `  #超话名称#`）
- `pic_id`：逗号分隔的 pid
- `topic_id`：超话 ID，一般通过搜索接口获取，也可以硬编码
- `location`、`visible` 等辅助字段

同样需要 Referer、X-CSRF-TOKEN 和完整 Cookie。成功返回 `ok: 1` 及帖子 ID。遇到 `ok: 0` 且 errno 为 `10022` 或 `10023` 通常意味着限流，需要等待。

### 4. 限流处理策略

我们没有采用固定间隔，而是在 MCP 工具内缓存上一次发帖时间戳，并实现简单的令牌桶 + 指数退避：

- 若返回 `10022`（操作频繁），等待间隔 = `min(60, 2^n + random(1,5))` 秒，n 为重试次数
- 在 `publish_status` 内部维护速率，连续成功发帖之间间隔至少 15 秒
- 将限流信息向上暴露给 Agent，Agent 可以在流程中主动调用 `check_quota` 决策是否延后执行

### 5. 状态归档与去重

使用 SQLite 存储每次同步的摘要表：

```sql
CREATE TABLE sync_log (
    id INTEGER PRIMARY KEY,
    content_hash TEXT,
    status TEXT,  -- pending/success/failed/rate_limited
    weibo_mid TEXT,
    retry_count INTEGER,
    created_at TIMESTAMP
);
```

每次发布前，Agent 计算内容的 SHA256 摘要，查询是否已存在成功记录，实现业务级去重。发布成功后更新 `weibo_mid` 和状态；失败则记录错误原因与重试次数，当重试超过上限时人工介入。

## 踩坑记录

- **图片上传 302 重定向**：多数因为缺少 Referer 或 Cookie 中缺少 `SUB`，注意登录态的保鲜，可每 24 小时刷新一次 Cookie。
- **CSRF token 失效**：发帖接口偶发 `400`，需要重新从页面抓取 token。实践中我们用一个定时任务（每 2 小时）访问微博首页，从 Cookie 和页面中提取 token 并缓存到文件中，MCP 工具运行时读取。
- **超话 ID 的获取**：不能依赖固定的前端渲染格式，推荐使用 `https://weibo.com/ajax/community/topic?q=超话名称` 接口搜索，解析返回的 `topic_id`。
- **重复发帖**：除了业务去重，还要注意重试时 pid 不能重复使用。若发帖失败但图片已上传，下次可复用 pid，但应避免同一 pid 被多次提交至新帖子，可能需要标记已用。

## 可复用建议

- **封装为可组合的 MCP 工具**：将图片上传、发帖、查询等作为独立工具，让 Agent 按需调用，比写死脚本更灵活。例如，当检测到图片上传失败时，Agent 可自动降级为纯文字帖。
- **状态机驱动重试**：不要只依靠简单的循环重试，用状态字段驱动，可避免进程重启后丢失重试计数。
- **Cookie 管理独立化**：用单独的 cron 进程维护登录态，输出到共享文件，MCP Server 只读，从而避免并发刷新导致的互踢。
- **在 OpenClaw Agent 的规划步骤中加入“速率检查”**：让 Agent 在决定发帖前显式地调用 `check_quota` 工具，降低被风控的概率。

## 总结

在 OpenClaw 框架下，将微博超话同步抽象为一组 MCP 工具和 Agent 决策流程，能够显著提升自动化的可靠性和可维护性。图片上传的协议细节、基于错误码的自适应退避以及本地状态归档是三个不可或缺的基石。整套方案在生产环境稳定运行数月，日均同步数百帖，人工干预率低于 2%。所有工具实现均不依赖官方 SDK 的复杂封装，而是直接通过 HTTP 调用，提高了透明度和排障效率。

如果你的 OpenClaw 项目也有类似的多媒体平台同步需求，上述实践或许能少走一些弯路。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-13/732888c32210d93f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-13/969da7e89306257c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-13/6d3eb670713e0437.png)

