---
title: 自动发帖流水线的本地归档实践：文章、图片与 assets.json 的持久化方案
feedId: 29813
source: 综合讨论
publishedAt: 2026-07-20
---

## 背景

在构建基于 OpenClaw / Agent / MCP 的自动发帖流水线时，我们往往把精力放在内容生成和多平台分发上。流水线产出 Markdown 文章、AI 生成的配图，然后推送到各个渠道。但如果没有本地归档，这些内容本质上只是一次性产物——图片链接可能几天后就失效，标题、标签等元数据散落在各个平台的响应里，后续想做内容复用、回溯分析或二次编辑时会非常痛苦。

一个完整的自动化内容系统，不应该只负责“发出去”，还需要可靠地把“生产了什么”完整地保存下来。也就是说，我们需要在流水线里增加一个归档节点，将每一篇文章及其关联的图片、元数据，以可离线使用的方式持久化到本地。

## 问题分析

常见流水线的归档做法是直接把生成的 Markdown 文件扔进某个目录，然后加一条 git commit。这个方案存在几个明显缺陷：

1. **图片还是远程引用**：如果图片是临时生成的（比如通过 AI 绘图工具或截图服务），外部 URL 可能在几分钟到几天内失效，导致历史文章变成“图片裂开”的状态。
2. **元数据缺失**：只有 Markdown 正文，却没有发帖时用的标题、标签、发布状态、平台 ID 等信息，将来想重新发布或做数据统计时，需要重新解析或手动补全。
3. **单文件承载能力有限**：当一篇文章有多张图片，或者需要附带其他资源（例如视频缩略图、结构化数据导出）时，单文件组织形式会很快变得混乱。

理想的归档方案，应该像前端构建工具的静态资源管理一样：每个帖子自成一个独立文件夹，包含文章正文、所有图片的本地副本，以及一份结构化的元数据文件。这个文件夹可以随时打包、迁移、或者作为数据源导入到其他系统，不依赖任何外部服务。

## 设计方案：一文一夹 + assets.json

我们为每篇文章创建如下结构的归档单元：

```
posts/
└── 2025-04-14-my-post-slug/
    ├── index.md
    ├── image_01.png
    ├── image_02.jpg
    └── assets.json
```

- `index.md`：文章正文，其中所有图片引用已替换为相对路径（`./image_01.png`）。
- `image_*`：从原始远程 URL 下载下来的图片，命名规则简单可控，如 `image_{序号}.{扩展名}`。
- `assets.json`：结构化元数据文件，包含文章标题、slug、创建时间、平台发布信息、标签、原始图片 URL 与本地文件名的映射等。

这种模式的好处是：无论是人类查看还是程序处理都很直观；目录整体可复制、压缩、上传到其他存储而不丢失任何依赖；`assets.json` 可作为其他自动化流程的“索引入口”。

## 实现步骤（以 MCP 工具为例）

如果你的流水线是基于 OpenClaw 和 MCP 搭建的，可以将归档功能封装为一个 MCP tool，便于在 Agent 工作流中调用。下面给出核心逻辑（Node.js 风格伪代码）：

### 1. 创建归档目录

```ts
import { mkdir, writeFile } from 'fs/promises';
import { join } from 'path';
import slugify from 'slugify';

const baseDir = './posts';
const slug = slugify(title, { lower: true, strict: true });
const dirName = `${new Date().toISOString().slice(0,10)}-${slug}`;
const postDir = join(baseDir, dirName);
await mkdir(postDir, { recursive: true });
```

### 2. 下载图片并替换 Markdown 引用

需要用流式下载，避免大图撑爆内存，同时加上重试逻辑：

```ts
import axios from 'axios';
import { createWriteStream } from 'fs';
import { pipeline } from 'stream/promises';

const imageMap: Record<string, string> = {}; // {原始URL: 本地文件名}
let index = 1;

// 从 markdown 中提取所有图片URL (支持两种常见格式)
const imgRegex = /!\[.*?\]\((.*?)\)|<img\s+src=["'](.*?)["']/gi;
const urls = Array.from(markdown.matchAll(imgRegex), m => m[1] || m[2]);

for (const url of urls) {
  if (imageMap[url]) continue;
  const ext = url.split('.').pop()?.split('?')[0] || 'png';
  const localName = `image_${String(index).padStart(2, '0')}.${ext}`;
  const localPath = join(postDir, localName);
  
  // 下载，最多重试2次
  for (let attempt = 0; attempt < 3; attempt++) {
    try {
      const response = await axios({ url, responseType: 'stream', timeout: 10000 });
      await pipeline(response.data, createWriteStream(localPath));
      break;
    } catch (e) {
      if (attempt === 2) throw new Error(`下载失败: ${url}`);
      await new Promise(r => setTimeout(r, 2000));
    }
  }
  imageMap[url] = localName;
  index++;
}

// 替换 Markdown 中的图片链接为相对路径
let finalMarkdown = markdown;
for (const [url, localName] of Object.entries(imageMap)) {
  finalMarkdown = finalMarkdown.split(url).join(`./${localName}`);
}
```

### 3. 生成 assets.json 并写入文章

```ts
const assets = {
  title,
  slug,
  created_at: new Date().toISOString(),
  platforms: ["platform-a", "platform-b"], // 发布到的平台标识
  tags: ["automation", "archiving"],
  images: Object.entries(imageMap).map(([original_url, local_name]) => ({
    original_url,
    local_name
  }))
};

await writeFile(join(postDir, 'assets.json'), JSON.stringify(assets, null, 2));
await writeFile(join(postDir, 'index.md'), finalMarkdown);
```

### 4. 集成到流水线

在 OpenClaw 的工作流中，可以把这个逻辑封装成一个 tool 或一个 Skill。当内容生成完毕，准备发布前，调用 `archive_post` tool，传入 `title`、`markdown`、`platforms`、`tags` 等参数，由该 tool 完成本地归档后再继续发布流程。这样即使发布失败，文章也已经在本地被安全保存。

## 踩坑与注意事项

- **图片链接的正则覆盖不全**：部分 Markdown 中可能出现带查询参数的图片 URL，或者 HTML 写法中的单引号、跨行等情况。建议使用专门的 Markdown AST 解析器（如 `unified` + `remark`）来提取图片节点，避免正则遗漏。
- **文件名冲突**：slug 虽然可以保证唯一性，但在高并发或短时间内重复标题时，仍然可能撞车。可添加毫秒级时间戳或 UUID 后缀。
- **大文件性能**：当文章包含大量高分辨率图片时，下载和写入操作可能阻塞事件循环。在 Node.js 中应保持异步，并使用 `stream`；在 Python 中同理，使用 `aiofiles` 或线程池。
- **assets.json 版本管理**：随着业务演进，元数据结构可能变化。建议在 JSON 中增加 `version` 字段，方便日后做兼容解析。
- **跨平台可用性**：本地归档的最终目的是不依赖线上资源，所以一定要确保 `index.md` 里的图片引用为纯相对路径，不要写 `file://` 或绝对路径。

## 可复用建议

1. **将归档工具独立成 MCP 包**：提供一个 `post-archiver` MCP server，暴露 `archive_post` 工具，接收标准化参数。这样任何 Agent 或流水线都可以直接通过 MCP 协议调用，无需重复实现。
2. **与 git 搭档使用**：归档后自动执行 `git add` 和 `git commit`（带好 commit message），形成内容变更历史仓库。`assets.json` 中的时间戳可作为 commit 时间对齐。
3. **考虑增量归档**：如果流水线运行在 CI 环境中，每次只归档新增或修改的文章，避免重复下载已有的图片。可以通过检查 `assets.json` 是否存在来判断。
4. **扩展为“内容资产平台”的基础**：这一目录结构将来可以直接对接静态站点生成器（如 Hugo、11ty），将归档文件直接渲染为个人博客或文档站，实现“一次归档，多处复用”。

## 总结

自动发帖流水线不应该把本地归档当成可选的附加项，而应作为核心数据资产化的一环。通过「一文一夹 + assets.json」的简单约定，我们可以保证：

- 文章在任何时候都可离线查看，图片不会丢失。
- 元数据结构化存储，为后续分析和二次利用提供可靠基础。
- 归档结果即静态资源，可直接被其他系统消费。

在工程实践中，这段归档逻辑代码量不大，但能显著提升整个自动化内容系统的鲁棒性和可维护性。如果你已经在用 OpenClaw 或 MCP 搭建内容流水线，建议立刻补上这个归档节点。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-20/50f82b43209b1980.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-20/56659b81e6b0d26c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-20/b55940914540bc0b.png)

