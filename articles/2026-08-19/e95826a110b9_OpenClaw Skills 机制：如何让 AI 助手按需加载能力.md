---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 33841
source: 综合讨论
publishedAt: 2026-08-19
---

# OpenClaw Skills 机制：如何让 AI 助手按需加载能力

## 背景

在 OpenClaw 的日常使用中，能力扩展通常靠 MCP server、脚本插件或外部 API 完成。早期我们会把文件处理、网页抓取、日历读写、消息推送、数据库查询等能力全部常驻。结果很快暴露问题：模型上下文被大量工具 schema 占满，启动变慢，工具选择准确率下降，甚至出现两个能力因同名工具冲突。

更关键的是安全边界：一个只做摘要的请求，不应该默认获得写文件或发消息的权限。我们需要让 AI 助手“知道很多能力存在，但只在实际需要时加载”。

这就是 Skills 机制要解决的问题。

## 问题拆解

全量加载能力主要有三个工程痛点：

1. **上下文膨胀**：每个 MCP 工具都有名称、描述、参数 schema。几十个工具常驻后，模型注意力被稀释。
2. **工具冲突与误选**：功能相近的工具容易混淆，比如 `get_page` 和 `fetch_url`。
3. **权限暴露**：常驻工具意味着模型随时可能调用，即便当前任务并不需要。

Skills 机制的核心思路是：**默认只保留轻量索引，按需加载完整实现**。

## 做法与步骤

### 1. 为每个能力写 Skill Manifest

每个 skill 是一个独立目录，包含描述文件、入口脚本或 MCP 配置。最小 manifest 可以这样写：

```yaml
name: web-capture
description: Capture a URL to Markdown or screenshot
when:
  keywords: ["capture", "screenshot", "web", "url"]
  requires: ["url"]
tools:
  - type: script
    entry: skills/web_capture/main.py
  - type: mcp
    server: playwright
    start: on-demand
permissions:
  - network: outbound
ttl: 600
fallback: return_error
```

这里的关键字段不是 `tools`，而是 `description` 和 `when`。它们决定 skill 能否被准确触发。

### 2. 建立轻量索引

OpenClaw 启动时只扫描所有 skill 的 `name`、`description` 和 `when`，生成一个索引。索引体积很小，可以放进系统提示或交给轻量 router 使用。具体工具定义和脚本内容不会被加载。

### 3. 匹配与按需加载

用户请求进来后，先做一次匹配。可以用关键词初筛，也可以用 embedding 计算语义相似度。命中后，OpenClaw 才执行加载动作：

- 注入该 skill 的 system prompt；
- 注册其声明的脚本工具或 MCP server；
- 按 `permissions` 限制权限范围。

例如用户说“帮我把这个网页截图存成 Markdown”，只有 `web-capture` 被加载，其他 skill 保持静默。

### 4. 生命周期管理

任务完成后，根据 `ttl` 卸载工具并释放上下文。若 skill 内部启动的 MCP server 支持优雅关闭，则先尝试 close；否则进行进程级回收。加载超时或执行失败时走 `fallback`，不要让主流程卡住。

## 踩坑点

**触发词太宽泛**  
`web`、`file` 这类高频词容易造成误触发。要观察触发命中率，必要时增加 `exclude` 条件，或在 `description` 里写清适用与不适用场景。

**卸载不干净**  
部分 MCP server 对 close 信号处理不完整，导致工具残留。建议统一做进程级回收，并在每次任务结束后检查活跃工具列表。

**多 skill 同时命中**  
一个请求可能同时命中 `web-capture` 和 `pdf-summarizer`。如果不加限制，上下文又会膨胀。可以设置 `max_active_skills`，按优先级只加载 top 1～2。

**远程 MCP 启动慢**  
按需启动的 MCP server 可能因网络或依赖问题拉取失败。必须配置启动超时和降级策略，例如直接返回“该能力暂不可用”，而不是无限等待。

**权限声明不等于系统隔离**  
manifest 里的 `permissions` 只是声明，真正安全还要依赖沙箱或系统权限。写操作类 skill 应默认要求用户显式确认。

## 可复用建议

- 用最小 manifest 校验器保证字段完整，尤其是 `ttl` 和 `fallback`。
- 给所有 skill 设置默认 `timeout: 30s`、`ttl: 300s`，避免资源长占。
- 将 skill registry 纳入 Git 版本管理，锁定依赖版本。
- 记录每次触发：skill 名、匹配耗时、执行耗时、是否成功。用于后续收紧路由。
- 将 skill 分为只读与写操作两类，写操作必须二次确认。
- 高频基础能力如 memory、clock 可保持常驻；低频重能力才走按需加载。

## 总结

Skills 机制的价值不在于“挂更多工具”，而在于“让模型在正确的时刻只看到需要的工具”。它能显著降低上下文消耗、减少工具误选，并提升安全边界。

实现的关键不是复杂的调度算法，而是三件事：**manifest 描述是否清晰、路由匹配是否准确、加载与卸载是否干净**。做好这三点，OpenClaw 的能力扩展才能真正走向工程化。

---

