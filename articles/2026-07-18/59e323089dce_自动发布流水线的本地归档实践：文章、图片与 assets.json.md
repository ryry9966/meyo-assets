---
title: 自动发布流水线的本地归档实践：文章、图片与 assets.json
feedId: 29564
source: 综合讨论
publishedAt: 2026-07-18
---

## 背景

在使用 OpenClaw 构建自动发帖流水线时，文章生成、配图生成、多平台分发通常会拆成多个 Agent 或步骤。一段时间运行下来，本地工作目录会堆积大量 Markdown 文件和图片，时间一长就分不清“这篇文章配了哪张图”“这张图被哪些文章引用过”。如果后续需要重新发布、修改配图或审计内容，往往要手工翻找，效率低且容易出错。

因此，在设计自动发布流水线时，可以加入一个**本地归档步骤**，把每次发布的文章、所用图片和元数据统一存入结构化目录，并用 `assets.json` 记录关联关系。这样即使发布完就不再关心，未来也能快速追溯，同时方便仓储管理和二次利用。

## 问题拆解

典型的自动发帖流水线流程：

1. Agent 调用 LLM 生成文章 Markdown 正文。
2. 另一个 Agent（或同一流程）生成封面图、内嵌插图，得到本地图片文件。
3. 发布步骤将 Markdown 转为平台格式，上传图片，完成发布。

在这个过程中，文章和图片的关联只存在于发布步骤的一次性映射里，发布完成后这些映射通常就丢失了。即便生成了本地文件，也常是散落在 `output/`、`images/` 等临时目录，没有归档索引。当你要：

- 修改某个已发布文章的配图
- 查看某张图片曾被哪些文章使用
- 换个平台重新发布时，快速找到原始文件和原图

都很麻烦。

## 归档方案设计

### 目录结构

以“发布批次”或“日期+主题”为维度创建归档目录，例如：

```
archive/
└── 2025-03-10-ai-agent-practice/
    ├── article.md
    ├── images/
    │   ├── cover.png
    │   ├── diagram.png
    │   └── workflow.png
    └── assets.json
```

`article.md` 是发布时的定稿内容，内部图片引用统一使用相对路径，如 `images/cover.png`。`images/` 存放该文章用到的所有图片原文件。`assets.json` 记录结构化的资源映射。

### assets.json 设计

推荐使用 JSON 格式，便于程序读取和人工查验：

```json
{
  "article": "article.md",
  "created_at": "2025-03-10T10:30:00Z",
  "platforms": ["medium", "dev.to"],
  "assets": [
    {
      "local_path": "images/cover.png",
      "role": "cover",
      "remote_urls": {
        "medium": "https://cdn.medium.com/...",
        "dev.to": "https://dev.to/..."
      }
    },
    {
      "local_path": "images/diagram.png",
      "role": "content",
      "remote_urls": {
        "medium": "https://cdn.medium.com/...",
        "dev.to": "https://dev.to/..."
      }
    }
  ]
}
```

这样既能通过 `assets.json` 反向查找某张图片被哪些平台使用，也能在本地打开 `article.md` 时直接预览图片。

### 集成到流水线

可以在发布步骤执行之前或之后，调用一个归档脚本。以 Node.js 为例，脚本大致逻辑：

1. 读取最终发布的 Markdown 文本和已上传后的远程 URL 映射。
2. 创建按日期命名的归档目录。
3. 将 Markdown 中所有远程图片 URL 替换为相对路径，然后写入 `article.md`。
4. 把本地生成的图片原文件复制到 `images/` 目录。
5. 生成 `assets.json`，填入本地路径与远程 URL 的对应关系。
6. 可选：打一个 Git commit 作为版本快照。

在 OpenClaw 的插件体系里，可以把这段逻辑封装成一个 `afterPublish` 钩子，或者作为独立 MCP 工具，让 Agent 在完成发布后主动调用。

### 具体实现片段

假设你已经有一个 `publishResult` 对象，包含文章内容、平台返回的图片 URL 和本地图片路径：

```javascript
const fs = require("fs-extra");
const path = require("path");
const dayjs = require("dayjs");

async function archivePost(publishResult) {
  const dateStr = dayjs().format("YYYY-MM-DD");
  const topicSlug = sluggify(publishResult.title);
  const archiveDir = path.join("archive", `${dateStr}-${topicSlug}`);
  await fs.ensureDir(path.join(archiveDir, "images"));

  // 替换内容中的远程图片为本地相对路径
  let localContent = publishResult.content;
  const assets = [];
  for (const img of publishResult.images) {
    const localFileName = path.basename(img.localPath);
    await fs.copy(img.localPath, path.join(archiveDir, "images", localFileName));
    localContent = localContent.replace(new RegExp(img.remoteUrl, "g"), `images/${localFileName}`);
    assets.push({
      local_path: `images/${localFileName}`,
      role: img.role || "content",
      remote_urls: img.remoteUrls || { [publishResult.platform]: img.remoteUrl },
    });
  }

  await fs.writeFile(path.join(archiveDir, "article.md"), localContent);
  await fs.writeJson(path.join(archiveDir, "assets.json"), {
    article: "article.md",
    created_at: new Date().toISOString(),
    platforms: [publishResult.platform],
    assets,
  }, { spaces: 2 });
}
```

这样每次发布后都会得到一个完整的归档快照。

## 踩坑点

1. **图片文件名冲突**
   不同文章可能生成相同名字的图片（如 `cover.png`）。归档时建议借助哈希或时间戳前缀重命名，或靠目录隔离已经足够。如果用的生成工具自动命名，可以直接沿用，但要在 `assets.json` 里保留原始别名信息。

2. **路径分隔符**
   Windows 和 Linux 路径混用容易出问题。归档时统一使用正斜杠 `/`，并在复制操作中使用 `path.join` 保证兼容性。

3. **大量图片场景**
   当一篇文章包含很多图片时，复制操作可能较慢。可以采用硬链接（`fs.link`）或仅在归档中记录引用，不复制实际文件，但这样会丢失历史独立快照，取舍要根据需求。

4. **远程 URL 不可预知**
   有些平台上传图片后返回的 URL 带有随机哈希，很难从本地文件名反向推断。所以必须在发布后立即收集远程 URL 写入 `assets.json`，而不是事后再爬取。

5. **归档与 Git 的协作**
   归档目录放入 Git 版本控制后，图片二进制文件会使仓库体积膨胀。建议为图片设置 Git LFS，或只归档 Markdown 和 `assets.json`，图片另存到对象存储并记录 URL。

## 可复用建议

- **通用脚本**：上述归档逻辑可以抽成一个 NPM 包或独立脚本，接受 Markdown 内容和图片映射作为输入，输出归档目录。不同平台的发布流程只需调用此工具即可。
- **与 OpenClaw 插件结合**：在自定义发布插件中添加 `postPublish` 钩子，直接将归档作为发布流程的最后一步，避免遗忘。
- **资产管理**：如果后续要查询“某张图被哪些文章使用”，可以维护一个全局索引，每次归档时将 `assets.json` 内容追加到总索引中，或用 SQLite 存储。
- **自动化测试**：每次归档后，运行快速检查：验证 `article.md` 里引用的 `images/xxx` 路径在 `images/` 目录下都存在，`assets.json` 中记录的本地文件也存在，减少“死链”归档。

## 总结

自动发帖流水线不只是“生成-发布”两个动作。加上一个轻量的本地归档环节，可以大幅提升内容的可维护性和复用性。通过约定好的目录结构和 `assets.json`，文章、图片和发布信息形成闭环，方便回头修改、平台迁移或团队协作。实现起来成本不高，却能让你的自动化内容生产系统更接近工程化标准。

---

