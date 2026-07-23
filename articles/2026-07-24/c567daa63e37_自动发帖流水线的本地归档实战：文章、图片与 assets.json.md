---
title: 自动发帖流水线的本地归档实战：文章、图片与 assets.json
feedId: 30254
source: 综合讨论
publishedAt: 2026-07-24
---

## 背景与问题

在基于 Agent 的自动化发帖流水线中，我们往往只关注“生成内容 → 推送到各平台”这条路径。一旦内容被顺利发布，很多人就认为任务结束。然而，当你需要回溯某篇文章的原始版本、审计某次发布失败的原因，或者将内容迁移到另一个平台时，缺少结构化本地归档会让你寸步难行。

常见痛点包括：

- 生成的 Markdown 和配图只存在于 Agent 的内存或临时目录，流水线结束后就丢失；
- 图片仍是远程 URL，离线无法查看，且链接随时可能失效；
- 发布元数据（发布时间、平台、帖子 ID 等）散落在各个平台的回调或日志里，难以统一检索；
- 多次手动补救，流程不可复现。

因此，我们需要在自动发帖流水线中加入一个 **本地归档步骤**，把文章、图片和发布元数据按约定结构持久化到本地文件系统，形成一套可追溯、可迁移的内容资产。

## 设计目标

1. **结构化存储**：每篇文章一个独立目录，内含 Markdown、图片及元数据文件。
2. **图片本地化**：自动下载远程图片，将 Markdown 中的图片链接替换为相对路径，保证离线可读。
3. **元数据索引**：生成 `assets.json` 记录文章的全部关键信息，便于程序消费。
4. **可复现、可集成**：归档过程作为独立步骤，能被 OpenClaw Agent 或 MCP 工具调用，不依赖特定平台。

## 归档方案设计

### 目录结构

```
archive/
└── 2025/
    └── 01/
        └── 15/
            └── my-article-slug/
                ├── index.md
                ├── images/
                │   ├── cover.png
                │   └── diagram-01.png
                └── assets.json
```

- 按日期分层，方便按时间检索。
- 文章 slug 作为目录名，保证唯一且可读。
- `assets.json` 作为该文章的元数据清单。

### assets.json 结构

```json
{
  "slug": "my-article-slug",
  "title": "Article Title",
  "created_at": "2025-01-15T10:30:00Z",
  "platforms": [
    {
      "name": "twitter",
      "post_id": "123456",
      "published_at": "2025-01-15T10:35:00Z",
      "url": "https://twitter.com/..."
    }
  ],
  "images": [
    {
      "local_path": "images/cover.png",
      "original_url": "https://cdn.example.com/img.png",
      "sha256": "abc123..."
    }
  ],
  "tags": ["automation", "archive"],
  "status": "archived"
}
```

可根据实际需要扩展字段，但至少应包含平台发布记录和图片映射关系。

## 具体实现步骤

### 1. 在 Agent 工作流中插入归档节点

以 OpenClaw 为例，在内容生成与发布之间（或发布成功后）调用归档 Action。可以将归档逻辑封装为一个 MCP Server，对外提供 `archive_post` 工具。

MCP 工具参数示例：

```json
{
  "markdown_content": "...",
  "images": [
    { "url": "https://...", "filename": "cover.png" },
    { "url": "https://...", "filename": "diagram-01.png" }
  ],
  "metadata": {
    "slug": "my-article-slug",
    "title": "...",
    "platforms": [],
    "tags": []
  },
  "archive_root": "/data/archive"
}
```

### 2. 实现归档核心逻辑

使用 Python 或 Node.js 编写归档函数，流程如下：

- 根据 `archive_root` 和当前日期、slug 生成目标目录 `{archive_root}/YYYY/MM/DD/{slug}/`。
- 创建目录，若存在则视为更新操作（需谨慎处理）。
- 下载所有远程图片，保存到 `images/` 目录，记录下载结果，并计算文件哈希以便去重。
- 修改 Markdown 内容，将图片链接替换为 `images/xxx` 相对路径。
- 写入 `index.md`。
- 生成或更新 `assets.json`：合并已有元数据（如果归档更新），记录平台发布信息、图片列表等。
- 返回成功信息或错误详情。

### 3. 集成到流水线

如果你的发帖流水线已经跑在 GitHub Actions 或类似 CI 环境，可以将上述 MCP 工具部署为独立服务，或者直接在 Action 中调用归档脚本。确保执行环境有写入归档目录的权限，且能访问外部图片资源。

## 踩坑与解决方案

1. **图片下载失败或超时**  
   - 使用重试机制（最多 3 次），并记录失败状态，在 `assets.json` 中标记为 `download_failed`，不阻塞整体流程。

2. **并发归档导致 assets.json 损坏**  
   - 如果多个 Agent 同时操作同一 slug，需要加锁。可以使用文件锁（`fcntl`/`lockfile`）或者架构上保证每个 slug 单线程写入。

3. **远程图片链接失效**  
   - 归档时校验 HTTP 状态码，失败则保留原始链接，警告但继续。后续可设置定时任务检查并补下载。

4. **文件名冲突**  
   - 下载时若检测到同名文件，根据 URL 哈希或序号重命名，并更新 Markdown 引用。

5. **时区混乱**  
   - 所有时间戳统一使用 UTC，归档目录日期也使用 UTC 日期，避免夏令时等问题。

## 可复用建议

- **将归档功能做成可配置的 MCP 工具**，不仅能被 OpenClaw 调用，也可以被任何支持 MCP 的 Agent 使用，提升复用性。
- **利用 assets.json 构建索引页或 RSS 源**，把归档目录变成一个轻量级内容库，方便检索或再次发布。
- **周期性扫描归档目录**，检查是否有 `index.md` 缺少图片，或 `assets.json` 中记录了失败资源，然后自动触发修复流程。

## 总结

在自动化发帖流水线中加入本地归档步骤，看似多了一层操作，但它为团队带来了长期的内容资产积累、故障排查能力和平台迁移自由度。通过统一的结构与元数据，我们将零散的发帖输出变成了可管理、可编程的数字资产。

对于正在使用 OpenClaw、Agent 或 MCP 做自动化实践的工程师，不妨立刻在你的流水线中补上这一环——哪怕最初只是一个简单的 shell 脚本，也能在日后省去大量“当初那篇文章去哪了”的痛苦。

---

