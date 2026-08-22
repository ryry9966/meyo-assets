---
title: 从零搭一个 AI 社区发帖机器人：OpenClaw + MCP 的落地记录
feedId: 34225
source: 综合讨论
publishedAt: 2026-08-22
---

## 背景

我们在维护一个技术社区时，每周需要发布若干篇公告、教程更新和资源汇总。人工处理不仅耗时，还容易漏发、格式不统一。于是尝试用 OpenClaw 搭一个“发帖机器人”，目标不是完全无人化，而是把重复的格式整理、草稿生成和发布动作自动化，保留人工审核。

实际做下来发现，真正难的不是“让模型写一段帖子”，而是发布链路本身。

## 问题

刚开始以为只是接一个生成接口，再调一下论坛 API。实际遇到的约束有：

- 论坛需要登录态、CSRF 校验和限流；
- 模型输出不能直接进正式版块，需要草稿和审核；
- 失败重试要有状态，不能重复发帖；
- 定时任务要可观测，否则半夜挂了没人知道。

这些问题决定了架构不能做成“Agent 直接调用发布工具”的简单链路。

## 做法与步骤

### 1. 架构拆分

我把流程拆成四块：

- **Generator**：用 OpenClaw Agent 生成 Markdown 草稿；
- **Reviewer**：规则过滤 + 人工确认；
- **Publisher**：MCP server 封装论坛 HTTP 接口；
- **Watcher**：记录日志、状态和失败告警。

OpenClaw 在这里主要负责内容生成和工具调度，不直接持有发布权限。

### 2. 把论坛接口封装成 MCP

实现一个 `mcp-forum-publisher`，暴露四个工具：

- `draft_post`
- `publish_post`
- `check_login`
- `list_posts`

内部用 `requests.Session` 维持登录，每次发帖前从页面解析 CSRF token，再带 cookie 提交。`draft_post` 只写草稿箱，`publish_post` 需要显式调用。

### 3. 限制 Agent 的权限

在 OpenClaw 的工具配置里，只把 `draft_post` 开放给生成链路；`publish_post` 放在单独执行器中，由人工在后台确认后触发。这样即使 prompt 被诱导或生成结果有偏差，最多污染草稿箱，不会直接发布。

### 4. 发布执行与重试

发布器使用简单队列，每个任务带 `idempotency_key` 和 `max_retries=3`。遇到 429 做指数退避，遇到 401/403 停止并告警，不盲目重试。发布成功后记录 `forum_post_id`，便于回溯。

## 踩坑点

### 别让 Agent 直接发布

最初为了方便，把 `publish_post` 也开放给了 Agent。结果一次生成内容里包含“请帮我把这条发到公告区”，Agent 真的调用工具发到了正式版块。后来改成“生成 + 草稿 + 人工确认”三段式，发布工具单独授权。

### 登录态比想象中脆弱

论坛 cookie 可能几小时就失效，尤其是 MCP server 重启后 session 丢失。解决方式是：定时 `check_login`，cookie 落盘到本地文件或 Redis，发布前强制检查登录态。

### 限流重试会加重问题

有一次遇到 429 后重试间隔太短，导致账号被临时限制。后来改成站点级 QPS 限制 + 指数退避，失败任务进死信队列人工处理，而不是无限重试。

### Markdown 格式不通用

很多论坛不是标准 Markdown，代码块、表格容易乱。后来在 `publish_post` 里做格式转换，并限制 Prompt 只允许标题、段落、列表、链接、代码块这些基础元素。

### 时区与定时发布

调度器用 UTC，论坛显示本地时间，曾导致帖子“提前”发布。统一在配置里写清发帖时区，并在发布日志里同时记录 UTC 和本地时间。

## 可复用建议

- 发布类自动化先做 dry-run，至少先落到草稿箱；
- 发布工具与生成工具权限隔离，不要给 Agent 最终发布权；
- 每个任务带幂等键，防止重复发帖；
- 重试区分 4xx 和 5xx，4xx 不要硬重试；
- 日志带 `trace_id`、`post_id`、`forum_post_id`，便于排查；
- 论坛接口封装成 MCP 后，先写 smoke test，再交给 Agent 调用。

## 总结

这个发帖机器人稳定运行后，主要节省的是格式整理、草稿生成和重复发布动作。真正的发布决策仍保留在人工侧。工程上最值钱的不是模型能力，而是把“不可靠的模型输出”和“不可靠的论坛接口”隔离开，中间加一层可审计、可回滚的控制。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/19205d0f8e2e7721.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/e916a3ae247560b8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/60521bad2ee67ae6.png)

