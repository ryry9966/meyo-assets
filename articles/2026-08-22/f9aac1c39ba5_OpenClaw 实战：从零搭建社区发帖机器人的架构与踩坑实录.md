---
title: OpenClaw 实战：从零搭建社区发帖机器人的架构与踩坑实录
feedId: 34202
source: 综合讨论
publishedAt: 2026-08-22
---

## 背景

在 OpenClaw 社区，维护者或重度用户常需定期发布技术摘要、资源汇总或版本更新。这类内容结构固定，但手动排版、发布耗时。我尝试用 OpenClaw 的 Agent 能力做一个“社区发帖机器人”，目标不是完全无人值守，而是把重复劳动降到最低：自动采集、生成草稿、人工审核、一键发布。

## 问题

社区没有官方发帖 API，只能模拟 Web 操作。登录态、反爬、内容质量、发布稳定性都需要自己处理。最大风险是生成低质或幻觉内容污染社区，因此设计上必须保留人工审核。

## 架构

整体分三层：

- 数据源层：RSS、GitHub Releases、CHANGELOG。
- Agent 编排层：OpenClaw 负责调度、提示词约束、工具调用。
- 执行层：两个 MCP 工具——`http_client` 处理请求，`playwright` 处理登录和复杂交互。状态存 SQLite。

流程：定时采集 → 清洗去重 → 生成草稿 → 存入 SQLite → 通知审核 → 调用发布工具 → 记录结果。

## 做法步骤

1. 抓包分析社区发帖接口，确认 `POST /posts` 的 body 结构、CSRF token 位置、需要的 header。
2. 注册 MCP server。`requests` 工具直接调 HTTP；`playwright` 用于首次登录和 Cookie 刷新。避免用浏览器自动化做所有事，太脆。
3. 在 OpenClaw agent 配置里把 `temperature` 调到 0.2，提供事实源和固定模板，要求生成内容时只使用给定链接。
4. 写发布工具 `publish_post`，输入 markdown，内部转成社区格式（HTML 或 BBCode），自动添加标签和来源。
5. 实现状态机：`draft -> review -> approved -> publishing -> published / failed`。只有 `approved` 才能调用发布。
6. 用 cron 每天触发采集生成，输出草稿到审核队列。

## 踩坑点

- **Cookie 有效期短**：登录后 Cookie 大概 12 小时失效，硬编码进环境变量第二天就 401。改为每次发布前用 `playwright` 检查登录态，失效就自动重登。
- **接口返回 200 但不是真发布**：社区可能会把帖子标记为“待审核”或直接丢弃，但接口返回 200。需要发布后查询帖子列表确认是否可见。
- **Markdown 转格式丢换行**：代码块和列表在社区编辑器里会挤在一起。转换时对代码块做了保留空白处理，列表前补空行。
- **频率限制**：同一账号短时间内发多条会触发验证码。加入随机 3-8 分钟延迟，每天最多发 2 条。
- **内容幻觉**：虽然提示词要求只用给定链接，但 Agent 偶尔会编造不存在的 URL。生成后加了一个校验步骤，用 `http_client` 检查所有外链返回码。
- **Playwright 选择器脆弱**：社区改版后按钮 class 变了，发布失败。后来将登录和发布拆开，发布优先走 HTTP API，只有登录用浏览器。
- **时区问题**：cron 容器跑在 UTC，发布时间总是早 8 小时。统一转成 `Asia/Shanghai`。
- **图片上传**：上传接口需要 multipart 和 CSRF token，且 token 与 Cookie 绑定。复用登录后的 session，并禁用了自动重定向。

## 可复用建议

1. 把发布动作封装成 MCP tool，而不是 agent 直接操作 DOM。接口稳定，也方便 mock 测试。
2. `dry-run` 模式必须有：只生成草稿和预览，不实际发布。新人调试时先跑这个。
3. 每个草稿和发布记录都带上 `trace_id`，失败时能对齐日志。
4. 所有凭证放环境变量，不要写进 agent 配置或数据库。
5. 日志记录请求方法、URL、状态码和响应摘要，不记录完整 Cookie。
6. 准备一个隔离的小号做测试，封号不心疼。
7. 人工审核是底线，尤其是发往公共社区，不建议全自动。

## 总结

这个机器人跑了两个月，从最初的完全自动失败，到现在的半自动稳定，核心变化是：降低模型自由度、增加发布后确认、保留人工审核、加强可观测性。OpenClaw 的 MCP 编排让这些模块可以灵活组合，但自动化发布最大的难点不在 AI，而在平台兼容和边界情况处理。如果你的需求是省时间，这个方案可行；如果想完全无人值守，建议先评估平台容忍度。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/99771dc07d6518c9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/80516789b95696f6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/99b95b46520ded8e.png)

