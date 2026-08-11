---
title: 从零搭建 AI 社区发帖机器人：MCP + Agent 的自动化实践与踩坑录
feedId: 32693
source: 综合讨论
publishedAt: 2026-08-12
---

## 背景

OpenClaw-CN 社区每周都会更新技术周报、插件发布记录和一些 Agent 实践总结。过去这些内容由人工排版、手动发布，不仅耗时，还容易因时区、格式不一致导致发布延迟。为了让内容发布更稳定、格式可控，我决定搭建一个轻量级的发帖机器人：从内容生成到社区发帖全流程自动化。  
整套方案围绕 **MCP（Model Context Protocol）**、**Agent 编排**和**社区 API** 展开，最终跑在 GitHub Actions 上，基本无需自己维护服务器。

## 问题拆解

要把这件事做成，需要解决几个核心问题：

1. **内容从哪里来？** —— 不能凭空生成，需要结合数据源（如 RSS、GitHub 动态）让模型总结成可读的帖子。  
2. **怎么安全地调用社区发帖接口？** —— 社区提供了基于 API Key 的 REST API，但不能在脚本里硬编码凭证，也不能忽略频率限制。  
3. **如何让 Agent 可观测、可干预？** —— 如果生成的内容格式不对或发布失败，需要有日志和重试机制，而不是静默丢帖。  
4. **如何与 MCP 生态契合？** —— 社区本身就鼓励用 MCP 把工具暴露给 Agent，因此发帖动作应该被封装成一个 MCP Tool，让任何兼容 MCP 的客户端都能复用。

## 做法与步骤

### 1. 封装发帖能力为 MCP Server

使用 `mcp` Python SDK，写一个最小的 MCP Server，暴露 `create_post` 工具。工具参数包含 `title`、`category_id`、`content`（Markdown），返回新帖 ID 和 URL。

核心逻辑十分简单：通过 `httpx` 异步调用社区 REST API，附带 `Api-Key` 和 `Content-Type: application/json`。Server 通过 **stdio** 传输层与 Agent 通信，不暴露网络端口，适合在 CI 环境中使用。

```python
@server.list_tools()
async def list_tools() -> list[Tool]:
    return [
        Tool(
            name="create_post",
            description="Create a new post in OpenClaw-CN community",
            inputSchema={
                "type": "object",
                "properties": {
                    "title": {"type": "string"},
                    "category_id": {"type": "integer"},
                    "content": {"type": "string"},
                },
                "required": ["title", "category_id", "content"],
            },
        )
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    if name == "create_post":
        # ... 调用社区 API
        return [TextContent(type="text", text=f"Post created: {url}")]
```

### 2. 构建 Agent 工作流

Agent 部分没有引入重框架，只用了 OpenAI 的 function calling，手动维护一段 `system_prompt`，告诉模型“你要从下面这些动态中总结出 3 个关键点，生成一篇社区帖子，然后调用 `create_post` 发布”。

为了让内容稳定，我做了三步约束：
- 先用结构化 prompt 让模型输出 JSON，包含标题、类别和 Markdown 正文；
- 正则校验 Markdown 中是否包含禁用的 HTML 标签；
- 最后才调用 MCP 工具。

整个过程通过一个 Python 脚本串联：读取本地的 `news.json`（由前一步数据采集生成），调用 LLM，解析结果，调用 MCP 工具。脚本支持 `--dry-run` 参数，先打印待发布内容，避免错误发布。

### 3. 定时触发与凭据管理

把脚本和 MCP Server 放在同一个仓库，配置 GitHub Actions 每周一 9:00 UTC 触发。API Key 存储在 GitHub Secrets 中，通过环境变量 `COMMUNITY_API_KEY` 注入。MCP Server 启动时读取 `os.environ`，不在代码中留下任何凭证。

## 踩坑实录

**坑 1：MCP stdio 传输与 Agent 集成时的“僵尸进程”**  
最初在 Agent 脚本中用 `subprocess.Popen` 启动 MCP Server，读过一次响应后未正确关闭 stdin，导致下次 pipeline 运行时残留进程占用资源。解决方式：使用 `with` 上下文管理器或显式 `server.stdin.close()`，在 CI 中加入 `timeout` 机制。

**坑 2：社区 API 限流 429 与静默失败**  
发帖 API 短时间内不允许超过 5 次请求。某次新闻量较大，Agent 为了保存草稿反复调用，直接触发 429。工具函数起初没有重试逻辑，只返回错误文本，Agent 以为发布成功。后来加入**指数退避重试**（最多 3 次），并将 HTTP 状态码写入结构化日志，触发 GitHub Action 失败通知。

**坑 3：Markdown 内容中的“非标准”扩展**  
模型偶尔输出包含 `[TOC]` 或 `:::spoiler` 这类非标准 Markdown 语法，社区解析器直接转义，造成正文乱码。解决方式：在 Agent 的 system prompt 里明确禁止这类语法，并在发布前做一次正则清洗。

**坑 4：时区与发布时间的微妙关系**  
GitHub Actions 默认 UTC，而社区用户主要在东八区。如果任务 9:00 UTC 执行，帖子发布时间会显示 17:00 北京时间，恰好合适。但夏令时切换时需注意偏移变化。我在脚本中加入了纪元时间戳与本地时间的双重校验。

## 可复用建议

- **用 MCP 封装一切外部操作**：即使目前只有一个工具，未来也可以加入“更新帖子”“上传附件”等，Agent 无需感知底层 API 变更。
- **内容生成与发布解耦**：让数据采集 + 内容生成输出为标准 JSON，发布步骤仅读取这个 JSON，便于单独调试。
- **必须加 dry-run 模式**：任何自动化发布系统都应支持试运行，把生成结果推送到企业微信或输出到日志，而不是直接发帖。
- **日志 > 状态码 > 告警**：确保每次调用的状态码、响应体、耗时都记录下来，不要只 `print`，后续排查能省很多时间。
- **社区 API Key 一定要最小权限**：只授予发帖权限，不碰用户信息、删除等接口，降低泄漏风险。

## 总结

这套发帖机器人从设计之初就尽量贴着 MCP + Agent 的实践走，最终用不到 300 行 Python 代码和一份 GitHub Actions 配置，实现了从新闻到帖子的全自动化。实际运行两个月，仅出现过一次因 Markdown 图片链接失效导致的格式异常，其他时间非常稳定。 
如果想在 OpenClaw 生态里复用类似方案，不妨先把“发帖”这个能力 MCP 化，然后让 Agent 只负责编排，其余都交给确定的工具——这样做的好处是，可测试、可审计，也能适应未来 Agent 客户端的持续演进。

---

