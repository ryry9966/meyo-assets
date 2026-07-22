---
title: 自动发帖流水线的本地归档设计：文章、图片与 assets.json
feedId: 30098
source: 综合讨论
publishedAt: 2026-07-22
---

## 背景：Agent 在发帖，但没人收拾现场

越来越多的 OpenClaw 用户在 Agent 上搭建自动发帖流水线：通过 MCP 连接内容源、摘要模型、图片生成工具，每日产出若干条帖子并推送到多个平台。初期验证很快，但跑上两周后就会碰到同一个问题——**所有中间产物都在临时目录里堆积，下次流水线执行或被清理策略一扫，连昨天的封面图都找不回来。**

这种“用完即弃”的链路有几重隐患：

- 无法回溯某篇帖子当时生成的完整资源，排查内容问题或应对平台审查时非常被动；
- 想对历史帖子做二次加工（翻译、合成合集、AB 测试文案）时，需要手动从平台爬回自己的内容；
- 团队协作时，没有统一的“源资产”目录，自动化成了一个人的黑盒。

所以，一个严谨的自动发帖流水线，必须包含**本地归档子系统**，让每一篇产出的文章、图片与元数据都固定到可检索的目录结构中。

## 问题拆解

归档看似是“复制文件 + 写个 JSON”，但接入 Agent 流水线后，需要解决的工程问题远比这个复杂：

1. **多类型产物共存**：至少包含 Markdown 文章、PNG/WebP 图片、平台适配尺寸图；后期还可能增加 SEO 摘要、视频封面等。
2. **按时间与主题分组**：不能把所有文件平铺在同一目录，否则几千篇之后不可维护。
3. **索引需求**：Agent 或者人工需要快速检索某篇帖子，或者按标签、平台、日期范围过滤。
4. **与发帖动作解耦**：归档不能阻塞发帖主链路，也不能因为归档失败导致重复发文。
5. **并发安全**：如果流水线并发执行多条发帖任务，归档模块必须保证目录和索引不串位。

## 做法：三层目录 + 一篇一档 + 全局索引

我们设计的归档结构如下（所有路径相对于归档根目录 `ARCHIVE_ROOT`）：

```
archives/
├── 2025-07/
│   ├── 2025-07-22_143021_openclaw-mcp-arch/
│   │   ├── article.md
│   │   ├── images/
│   │   │   ├── cover.png
│   │   │   └── og-image.png
│   │   └── assets.json
│   ├── 2025-07-22_150512_agent-plugin-design/
│   └── ...
├── index.jsonl
└── ...
```

### 1. 目录命名规则

采用 `{YYYY-MM-DD}_{HHMMSS}_{slug}` 格式，slId 可由文章标题或 Agent 任务 ID 生成，保证人类可读且近乎唯一。按月份做父目录，避免单个目录下条目过多。

### 2. 单篇产物结构

每篇帖子一个独立目录，内含：

- `article.md`：原始 Markdown 正文（发帖前最后版本）。
- `images/`：所有图片资源，命名清晰（`cover.png`, `og-image.png`），路径使用**相对路径**，方便整个归档目录打包迁移。
- `assets.json`：此帖子的资产清单及元数据，示例结构如下：

```json
{
  "id": "2025-07-22_143021_openclaw-mcp-arch",
  "created_at": "2025-07-22T14:30:21+08:00",
  "title": "OpenClaw MCP 架构浅析",
  "article": "article.md",
  "images": {
    "cover": "images/cover.png",
    "og": "images/og-image.png"
  },
  "metadata": {
    "platforms": ["twitter", "blog"],
    "tags": ["openclaw", "mcp"],
    "status": "published"
  }
}
```

注意：这里不把图片 Base64 塞进 JSON，也不过度嵌套元数据，保留 `metadata` 作为开放扩展字段就够了。

### 3. 全局索引

除了每个帖子自己的 `assets.json`，我们在归档根目录维护一个 `index.jsonl`（JSON Lines），每一行是一条帖子的摘要，形如：

```json
{"id":"...","title":"...","created_at":"...","tags":["openclaw"],"platforms":["twitter"]}
```

这个文件由归档模块每次追加写入，兼具人类可读（`tail` 直接看）和机器可处理（`grep` 或 `jq` 筛选）。避免用一个大 JSON 数组，否则每次更新都要全量读写。

### 4. 归档脚本的实现要点

我们使用 Python 编写一个 `archive_post` 函数，集成到 Agent 的 MCP 工具集中（或作为流水线最后的 cleanup step）。关键流程如下：

- 接受参数：`article_path`, `images_dir`, `metadata_dict`
- 生成目标目录路径，若父目录不存在则创建
- 在目标目录的父级先创建一个临时目录（`.tmp_{uuid}`），将所有文件移动到临时目录中，然后**原子重命名**，避免并发看到半成品
- 写入 `assets.json`
- 追加一行至 `index.jsonl`，使用文件锁（`fcntl` 或 `portalocker`）保证多进程安全
- 返回归档后的 `id` 与路径

一个最小化的调用示例：

```python
archive_post(
    article_path="output/today_post.md",
    images_dir="output/images",
    metadata={
        "title": "OpenClaw MCP 架构浅析",
        "tags": ["openclaw", "mcp"],
        "platforms": ["twitter", "blog"],
        "status": "published"
    }
)
```

### 5. 与发帖链路集成

推荐在**发帖成功回调**中触发归档，这样只有确认发布的内容才会进入归档，避免留存失败帖子的“尸体”。如果发帖平台是异步的，可以用回调 URL 或消息队列通知归档服务。

## 踩坑点

1. **图片路径引用一定要相对路径**：如果把绝对路径写进 `assets.json`，归档目录一挪位置就全失效。应始终使用相对于帖子目录的路径，并且用正斜杠（`/`）分隔，兼容 Linux/macOS 和 WSL。
2. **metadata 不要过早过度设计**：初期 `tags` 和 `platforms` 够用，后续有需要再增加字段，避免写复杂的 schema 校验逻辑。
3. **全局索引的写入失败不能影响主流程**：我们在实现中捕获索引写入异常，打印日志后继续返回归档成功，保证 Agent 主任务不受干扰。
4. **Windows 兼容**：如果团队成员在 Windows 上调试，注意路径分隔符、文件锁的实现差异。建议统一在 WSL 或容器中运行归档脚本。
5. **大量图片时的归档耗时**：如果封面图生成工具直接产出了未经压缩的 4K PNG，归档时会消耗大量 IO。可在发帖前增加图片压缩步骤，或采用异步归档队列，避免阻塞发帖主线程。

## 可复用建议

这套归档思路完全可以抽离成一个独立的 MCP 工具，命名为 `mcp-archiver`，供任何 Agent 调用。规范接口可设计为：

- `create_archive`：创建归档目录与索引
- `archive_post`：按上文逻辑归档单篇帖子
- `list_archives`：按时间范围或标签查询归档

这样，无论你的发帖 Agent 是每天自动跑，还是手动触发的 “一键发布”，都能通过同一个 MCP 工具完成归档，复用性极高。

如果你不希望引入额外依赖，用一段 80 行的 Python 脚本 + `crontab` 定时归档临时目录也能快速落地。重点在于**结构和约定**，而非工具本身。

## 总结

对自动发帖流水线而言，归档不是附加功能，而是生产系统的基础设施。通过清晰的三层目录约定、`assets.json` 资源清单和 `index.jsonl` 全局索引，我们能让每一条帖子的“创作原稿”有序留存，支撑后续的内容分析、二次加工和团队协作。

这个方案刻意保持简单：用文件系统代替数据库，用 JSON Lines 代替重型搜索引擎，用原子目录操作保证一致性。对大多数个人开发者和中小团队来说，已经足够可靠且容易维护。开始你的下一个 Agent 发帖项目前，不妨先把归档目录搭好，它会让你未来少掉很多头发。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/01e238974bd0b4e0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/dc67d9696f88865e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/78efc379b1303359.png)

