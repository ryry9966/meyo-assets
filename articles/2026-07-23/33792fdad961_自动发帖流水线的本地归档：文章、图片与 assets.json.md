---
title: 自动发帖流水线的本地归档：文章、图片与 assets.json
feedId: 30153
source: 综合讨论
publishedAt: 2026-07-23
---

## 背景

在以 OpenClaw、Agent、MCP 等工具驱动的自动发帖流水线里，我们通常会让大模型生成一篇完整的 Markdown 文章，再调用生图服务产生配图，最后推送到目标平台。一套跑下来，我们往往只能拿到平台上的发布链接和一份可能被平台改写过的最终文本。至于配图的原始 Prompt、图片文件本身、文章中间版本、生图参数等资产，如果没有刻意留存，随着临时 URL 过期或上下文被回收，就再也找不回来了。

这会带来两个直接问题：
1. **不可复现**：想回过头复盘为什么这个封面图效果好，或想在另一条流水线里复用同一套视觉风格，却发现 Prompt 和参数都已丢失。
2. **不可二次编辑**：如果你打算把自动化生成的内容整理成手册、独立站点或离线分享，缺少原始图片和结构化元数据会让整理成本非常高。

工程化的做法，是在发布前插入一个“归档节点”，把所有产物有组织地存到本地。

## 问题拆解

一个典型的自动化发帖流程可能产生如下资产：
- 文章 Markdown 正文（可能包含多处图片引用）
- 封面图（cover）
- 正文配图（若干张，每张对应不同段落）
- 生成每张图片时使用的自然语言 Prompt、模型名称、尺寸等参数
- 文章元信息（标题、Slug、标签、生成时间、使用的文本模型等）

如果只是在代码里随手把图片下载到一个临时目录，用时间戳命名，一旦目录结构混乱，后期回溯仍然困难。我们需要的是一种**自描述、可迁移的归档格式**，让任何一个拿到目录的人都能立刻理解里面的内容。

## 归档结构设计

一个比较实用的方案是：为每一篇文章建立一个以 `{YYYY-MM-DD}-{slug}` 命名的文件夹，内部固定包含三个要素：

```
archives/
└── 2025-03-17-how-to-benchmark-mcp-servers/
    ├── index.md
    ├── assets.json
    └── images/
        ├── cover.png
        ├── img-01.png
        └── img-02.png
```

- **index.md**：最终的 Markdown 文章，所有图片引用已替换为本地相对路径，例如 `![architecture](images/img-01.png)`。这意味着你可以用任何 Markdown 编辑器直接打开预览，所有图片都本机可见。
- **images/ 目录**：存放所有图片文件，命名规则按用途或顺序，避免随机字符串造成阅读混乱。
- **assets.json**：一份轻量级的资产清单，描述文章与图片的完整元数据。

`assets.json` 的一个参考结构如下：

```json
{
  "title": "How to Benchmark MCP Servers",
  "slug": "how-to-benchmark-mcp-servers",
  "created_at": "2025-03-17T10:30:00Z",
  "tags": ["mcp", "benchmark", "automation"],
  "text_model": "gpt-4o",
  "images": [
    {
      "file": "images/cover.png",
      "purpose": "cover",
      "prompt": "A minimalist isometric illustration of three server racks connected by glowing lines, clean tech style, no text",
      "model": "dall-e-3",
      "size": "1792x1024",
      "generated_at": "2025-03-17T10:30:15Z"
    },
    {
      "file": "images/img-01.png",
      "purpose": "body_diagram",
      "prompt": "Flowchart showing MCP client-server interaction with tool list and resource endpoints",
      "model": "dall-e-3",
      "size": "1024x1024",
      "generated_at": "2025-03-17T10:31:02Z"
    }
  ]
}
```

这个 JSON 的目的不是给机器做复杂自动化，而是**给人和未来的脚本一个清晰的索引**——一眼就能看出每张图是干什么用的、当时用了什么 Prompt 和参数。

## 在流水线中实现归档

实现的要点是把归档逻辑做成一个可独立调用的函数或 MCP 工具，插入到“内容已生成、即将发布”的环节之后。以 TypeScript 伪代码为例：

```typescript
async function archivePost(post: {
  title: string;
  slug: string;
  markdown: string;
  images: Array<{ url: string; purpose: string; prompt: string; model: string; size: string }>;
  tags: string[];
  textModel: string;
}) {
  const dir = `archives/${new Date().toISOString().slice(0,10)}-${post.slug}`;
  await fs.ensureDir(`${dir}/images`);

  const imageAssets = [];
  let md = post.markdown;

  for (let i = 0; i < post.images.length; i++) {
    const img = post.images[i];
    const filename = img.purpose === 'cover' ? 'cover.png' : `img-${String(i).padStart(2,'0')}.png`;
    await downloadImage(img.url, `${dir}/images/${filename}`);

    // 把 Markdown 中原始 URL 替换为本地路径
    md = md.replace(img.url, `images/${filename}`);

    imageAssets.push({
      file: `images/${filename}`,
      purpose: img.purpose,
      prompt: img.prompt,
      model: img.model,
      size: img.size,
      generated_at: new Date().toISOString()
    });
  }

  await fs.writeFile(`${dir}/index.md`, md);
  await fs.writeFile(`${dir}/assets.json`, JSON.stringify({
    title: post.title,
    slug: post.slug,
    created_at: new Date().toISOString(),
    tags: post.tags,
    text_model: post.textModel,
    images: imageAssets
  }, null, 2));
}
```

如果你的流水线基于 OpenClaw 的 Agent 编排，可以把这段逻辑抽象成一个 “local-archiver” 插件，作为后处理钩子挂载。如果用的是更松散的脚本串联，一个独立 Node.js 脚本就够用。

## 踩坑记录

1. **临时 URL 过期**  
   很多生图 API 返回的图片 URL 只有几分钟到几小时的有效期。归档必须在拿到 URL 后立刻下载，不能延后到发布成功之后。建议将下载图片和组装 assets.json 设计为原子操作——要么全部成功写入，要么整体清理重试。

2. **Markdown 替换不精确**  
   同一张图可能在文章中被引用多次（比如封面图既出现在 top 区域，又出现在“阅读更多”的缩略图里），简单的字符串替换可能导致漏替换或错误替换。更好的做法是在生成 Markdown 时就用占位符标识图片位置，归档时再填入真正的本地路径。如果做不到，则用完整的原始 URL 作为查找键，并注意 URL 编码问题。

3. **文件名冲突与路径兼容**  
   如果同一天内可能多次生成同一 slug 的内容（例如调试），要增加版本号或时间戳避免覆盖。路径统一使用正斜杠 `/`，保证 Windows 和 Linux 都能正常识别。

4. **assets.json 的手工可读性**  
   不要往 JSON 里塞过大的二进制内容或嵌套很深的配置，保持扁平。对 `prompt` 这样可能包含多行或特殊字符的字段，确保写入时正确转义，并用 UTF-8 无 BOM 编码。

## 可复用的工程建议

- **封装为独立工具**：把归档逻辑做成 CLI 工具或 MCP Server 的一个 method，输入是一个包含标题、slug、Markdown 正文、图片列表的 JSON 对象，输出是归档目录路径。这样可以在任何流水线中直接调用。
- **与元数据索引结合**：如果你同时维护一个本地 all-posts.json 或 SQLite 索引来管理所有归档文章，可以在归档函数里追加索引更新操作，避免手工维护索引。
- **定期校验**：写一个小脚本定期遍历 archives 目录，检查每个目录是否同时包含 index.md、assets.json 和 images 目录内所有图片，提前发现文件缺失。
- **CI/CD 集成**：如果你的发帖流水线本身跑在 GitHub Actions 或本地 cron 里，可以把归档目录整个推到一个私有 Git 仓库，形成自动版本控制。

## 总结

自动发帖流水线很容易陷入“生成-发布-忘记”的循环，但工程化的内容运营需要可回溯的资产沉淀。通过约定一个简单的本地归档结构——Markdown 文章 + 图片目录 + 一份 assets.json——我们能在不增加复杂依赖的前提下，把每次自动生成的完整上下文保留下来。这套方案对 Agent、MCP、自动化发布等场景都很友好，且能轻松集成进现有流水线，需要的只是“多一点下载和存盘”的工程意识。

---

