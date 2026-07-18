---
title: 自动发帖流水线的本地归档设计：Markdown、图片与 assets.json
feedId: 29601
source: 综合讨论
publishedAt: 2026-07-19
---

## 背景

在构建面向 OpenClaw、Agent 或 MCP 的自动内容发布流水线时，很多人的关注点都集中在“内容生成→多平台分发”上，却容易忽略一个工程基础：**本地归档**。没有归档，你会失去对批量内容的追溯能力，后续想要做二次编辑、数据分析或迁移时会非常痛苦。

我自己在维护的几个自动化发布流程里，遇到过这样的场景：一个 Agent 每天生成技术简报并推送到多个社区，运行一段时间后，需要查找某篇特定文章对应的配图，但配图早已被后来的临时文件覆盖；想批量提取所有文章的标题和 slug，却发现只有分散的日志，没有结构化的元数据。于是决定花精力把归档层做扎实。

## 问题拆解

自动发帖流水线的输出通常包含：

- 一篇 Markdown 正文（可能带 frontmatter）
- 0 到多张配图（由 AI 生成或从远程抓取）
- 一些关键元数据：发布时间、目标平台、标签、原始素材引用等

如果不加规范，这些零散文件会散落在临时目录、内存缓存甚至远程存储里。我们真正需要的是一个**可复现、可审计、可自动化**的本地归档方案，能够把一次发文任务的所有产物封装进一个独立的、自描述的子目录中，同时生成一份结构化的清单文件（如 `assets.json`）。

## 实践做法

### 1. 目录结构约定

我为每个发文单元设计了一个标准子目录：

```
posts/
  └── 2025-03-15-slug-here/
       ├── index.md
       ├── images/
       │    ├── cover.png
       │    └── diagram.png
       └── assets.json
```

- 目录名包含日期与 slug，保证按时间排序且可读。
- `index.md` 作为文章正文（内部图片引用统一使用相对路径 `./images/`）。
- 所有图片聚合在 `images/` 子目录中，无论它们原本来自远程下载还是本地生成。
- `assets.json` 记录完整的任务上下文，不依赖 Markdown frontmatter 解析。

### 2. assets.json 结构设计

我使用一个轻量的 JSON schema，核心字段如下：

```json
{
  "uuid": "a1b2c3d4-e5f6-...",
  "title": "Post Title",
  "slug": "post-title",
  "created_at": "2025-03-15T10:00:00Z",
  "tags": ["openclaw", "automation"],
  "source": {
    "type": "rss",
    "url": "https://example.com/original",
    "fetched_at": "2025-03-15T09:30:00Z"
  },
  "images": [
    {
      "file": "images/cover.png",
      "prompt": "A futuristic pipeline diagram...",
      "source": "generated",
      "alt": "Cover illustration"
    }
  ],
  "platforms": [
    { "name": "discord", "post_id": "123456", "posted_at": "..." }
  ]
}
```

这样做的价值：无论以后迁移到哪种数据库或静态站点生成器，这份 JSON 都能直接驱动索引或重建任务。

### 3. 集成到发帖流水线

在我的实现中（基于 OpenClaw 的自定义技能 + MCP 文件系统工具），发帖流水线的最后一步不是直接调用平台 API，而是**先执行本地写入**：

- Agent 生成 Markdown 后，通过 `write_file` 写入 `index.md`。
- 若文章含有图片生成步骤，则让图片模型输出到 `images/` 目录，并重命名为稳定文件名，同时更新 Markdown 中的引用。
- 用脚本或 Agent 工具函数收集上述元数据，序列化为 `assets.json` 写入同一目录。
- 完成本地归档后，再读取 `assets.json` 和 `index.md` 作为发布数据源，向各平台接口推送。

这样做还有一个好处：发布失败时，本地归档已经存在，重试不需要重新跑生成链路。

## 踩坑记录

**图片引用路径不一致**

最早我让 Agent 直接在 Markdown 里写 `![](cover.png)`，但图片实际放在子目录 `images/` 下，导致本地预览正常、推送到某些平台后图片失效。最后硬性规定：所有 Markdown 引用必须使用 `./images/` 前缀，归档脚本里做断言检查，否则报错修正。

**并发写入导致 assets.json 损坏**

当流水线同时处理多个发文任务时，如果直接对同一个 `assets.json` 做追加而非按子目录隔离，很容易出现写覆盖。采用每个任务独占一个子目录后，该问题自然消失。

**元数据与 frontmatter 双写不一致**

起初我把标题、日期等同时写在 Markdown frontmatter 和 `assets.json` 里，结果手工修改一个后另一个不同步。后来决定将 `assets.json` 作为唯一事实来源，Markdown 中不再保留冗余 frontmatter，或只保留渲染必需字段，其余从 JSON 注入。

**远端图片未本地化**

如果图片直接引用远程 URL 而不下载，归档就等于失效，源站删除后文章残废。现在流程中强制将远程图片下载到 `images/`，并更新 Markdown 链接，归档完整性得以保证。

## 可复用建议

1. **将目录结构打包成模板**：可以写一个简单的 CLI 或 MCP 工具，接受标题、slug、日期参数，生成完整子目录骨架，方便手工新建任务。
2. **加入一致性校验的 Git pre-commit hook**：例如检查每个子目录同时存在 `index.md` 和 `assets.json`，且图片文件与 JSON 中定义的列表匹配。
3. **用 Git 管理 `posts/` 目录**：每次发文归档后自动 commit，天然形成版本历史，方便回溯与协作。
4. **与 MCP 文件系统服务结合**：如果已经有 `mcp__filesystem` 可用，归档操作可以直接通过标准工具调用完成，无需额外依赖。
5. **生成索引页面**：基于所有 `assets.json` 聚合生成一个静态索引页，方便快速浏览历史文章，也能作为内部知识库入口。

## 总结

本地归档这件事，在自动发帖流水线里不难，但如果不从一开始就规范好，后续的维护成本会直线上升。通过固定目录结构、标准化 `assets.json`、强制图片本地化，以及把归档作为发布前的必要环节，我让整个自动化流程变得可回溯、可迁移、可扩展。对于 OpenClaw 用户来说，这其实也是 Agent 可靠性的一个体现——好的 Agent 不仅会发帖，还会把自己的产出管理得井井有条。希望这套做法能给你的自动化发布链路带来一些参考。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/6c2c892681b81a4b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/14b01dc3d3c5dfca.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/a799b732d02ab06e.png)

