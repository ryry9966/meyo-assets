---
title: 自动发帖流水线本地归档实战：文章、图片与 assets.json 的工程化落地方案
feedId: 28911
source: 综合讨论
publishedAt: 2026-07-13
---

## 背景

在搭建内容自动发布流水线时，很多同学的注意力都放在“如何把帖子发出去”这一环——对接平台 API、处理鉴权、格式化正文。但上线一段时间后会发现一个更棘手的问题：**发出去的帖子没有留底，后期校对、重发、合规审查、跨平台迁移时完全无迹可寻。**

对于已经在使用 OpenClaw、Agent 或 MCP 工具链的团队，通常希望流水线本身具备一定的“自归档”能力。也就是说，每一次自动生成的帖子及其关联的图片，都能在本地落盘一份结构化的存档，包括：

- 文章正文（Markdown 或纯文本）
- 下载到本地的图片（避免 CDN 链接失效）
- 一份 `assets.json`，记录这次发布涉及的所有资源及其元数据

这套归档机制不是“锦上添花”，而是生产级内容流水线和本地可观测性的基础。近期我在重构自己的发布 Agent 时，完整实现了一套本地归档流程，踩了不少坑，这里做一次记录。

## 问题拆解

看起来需求很简单：把文章存成文件，把图片下载下来，再生成一份 JSON 索引。但在实际工程中，会遇到三个典型痛点：

1. **图片链接生命周期不可控**  
   第三方图床、临时上传链接、带签名参数的 CDN 链接，短则数小时即失效。如果不及时下载到本地，后续归档等于白做。

2. **文章内部引用关系断裂**  
   很多自动生成工具默认输出的是远程图片 URL，归档后如果不替换成本地相对路径，Markdown 文件本身仍然依赖外网，无法离线查看。

3. **并发发布与归档的一致性**  
   如果 Agent 是异步并发的，短时间内可能产生多个帖子，如何保证每一条帖子、其图片和 assets.json 的严格对应，且不会因为任务失败导致脏数据？

下面是我最终采用的方案，全部基于 Node.js 实现，可以很方便地封装成 MCP 工具，或者集成到 OpenClaw 的 Tool 体系里。

## 做法与步骤

### 1. 定义归档目录结构

每次发布任务生成一个唯一 ID（可以使用 UUID 或时间戳+随机数），在本地创建一个以该 ID 命名的目录，比如：

```
archives/
  20250407-a1b2c3/
    post.md
    images/
      cover.png
      img_01.png
    assets.json
```

`post.md` 中是正文，所有图片引用已替换为 `./images/xxx`。`assets.json` 描述本次发布的所有资源。

### 2. 下载图片并生成本地文件名

核心函数：输入远程 URL，返回 `{localPath, localFileName}`。

```ts
async function downloadImage(url: string, destDir: string): Promise<{localFileName: string, localPath: string}> {
  const parsed = new URL(url);
  // 去参数、去哈希，提取尽量可读的文件名
  let rawName = path.basename(parsed.pathname) || 'image';
  // 保持唯一性
  const ext = path.extname(rawName) || '.png';
  const base = path.basename(rawName, ext).slice(0, 32); // 防止文件名过长
  const fileName = `${base}_${crypto.randomUUID().slice(0, 8)}${ext}`;
  const filePath = path.join(destDir, fileName);

  const res = await fetch(url);
  if (!res.ok) throw new Error(`Download failed: ${res.status}`);
  const buffer = Buffer.from(await res.arrayBuffer());
  await fs.promises.writeFile(filePath, buffer);
  return { localFileName: fileName, localPath: filePath };
}
```

**坑点 1：重名覆盖**  
即使文件名看起来不同，并发下载时如果只用原始文件名，还是会冲突。所以一定要加随机后缀。

**坑点 2：连接超时与重试**  
某些图床在大流量下会限速或临时不可达。建议用 `p-retry` 之类的库做 3 次重试，指数退避。

### 3. 替换文章内的图片引用

拿到 Markdown 正文后，用正则找出所有 `![...](url)` 和 `<img src="url">` 形式的引用，对每个 URL 调用 `downloadImage`，然后替换为 `./images/${localFileName}`。

这里要注意，同一个图片可能在文中出现多次，所以需要先做 URL 去重，避免重复下载。

```ts
const imgRegex = /!\[([^\]]*)\]\(([^)]+)\)/g;
const imgTagRegex = /<img[^>]+src=["']([^"']+)["'][^>]*>/gi;

const urlMap = new Map<string, string>(); // remoteUrl -> localFileName
// 收集所有需下载的 URL
let match;
while ((match = imgRegex.exec(markdown)) !== null) {
  urlMap.set(match[2], ''); 
}
// 对每个 URL 下载
for (const url of urlMap.keys()) {
  const { localFileName } = await downloadImage(url, imagesDir);
  urlMap.set(url, localFileName);
}
// 替换
let localMarkdown = markdown;
for (const [remote, local] of urlMap.entries()) {
  localMarkdown = localMarkdown.replaceAll(remote, `./images/${local}`);
}
```

**坑点 3：查询参数与 URL 规范化**  
同一个图片可能因为末尾多了 `?w=200` 而被识别为两个不同 URL，导致重复下载甚至替换失败。需要先对 URL 做一次规范化：去掉无关查询参数（如尺寸、token），仅保留标识资源本身的部分。这部分逻辑可以根据自己的图床规则定制。

### 4. 生成 assets.json

`assets.json` 不是简单的文件列表，它是这条帖子发布过程的“元数据快照”。建议至少包含以下字段：

```json
{
  "id": "20250407-a1b2c3",
  "createdAt": "2025-04-07T10:30:00Z",
  "platform": "openclaw-blog",
  "title": "xxx",
  "postFile": "post.md",
  "images": [
    {
      "originalUrl": "https://cdn.example.com/orig.png",
      "localFileName": "cover_a1b2c3d4.png",
      "altText": "架构图",
      "mimeType": "image/png"
    }
  ],
  "originalMarkdownUrl": null,
  "tags": ["automation", "archiving"]
}
```

其中 `images` 数组里的 `originalUrl` 可以用来追溯来源，即使日后 CDN 失效，也清楚这张图最初是从哪里来的。

### 5. 持久化

用 `fs.promises.writeFile` 把 `post.md` 和 `assets.json` 分别写入归档目录，整个归档过程就结束了。所有文件都在本地，不再依赖任何外部服务。

## 踩坑点总结

- **异步任务边界**  
  如果你的 Agent 用 BullMQ 或类似的任务队列，请确保“归档”是同一个 Job 的一部分，而不是另起一个异步任务。否则可能出现帖子发完了，归档还没来得及做，服务就挂了的情况，导致漏归档。

- **图片下载的并发控制**  
  一篇帖子里如果有十几张图，同时 `fetch` 可能触发服务端的速率限制。使用 `p-limit` 限制并发数（比如 5）会更友好。

- **磁盘空间与清理策略**  
  如果你只是短期归档，建议在归档目录上加 TTL 清理逻辑。如果是长期存档，最好另外同步到对象存储或本地 NAS，并做定期校验。

- **特殊编码的图片 src**  
  极少数编辑器生成的 `src` 可能是 Base64 data URI 而不是远程链接。这种情况可以选择直接解码写入本地，或者忽略，但要在 `assets.json` 里标明类型。

## 可复用建议

这套归档逻辑非常适合封装成一个 MCP 工具或者 OpenClaw 的 Tool 函数：

- **输入**：Markdown 正文、平台名称、帖子标题、标签（可选）
- **输出**：本地归档目录路径、`assets.json` 路径、已替换图片引用的 Markdown 文件路径

这样任何内容生成环节（无论是 AI 写作、排版、还是审核）都可以通过一次调用完成“就地存档”，而不必关心底层存储细节。

如果你正在设计自己的内容 Agent，我强烈建议在“发布”动作之前先强制调用归档工具。哪怕发布失败，至少内容已经安全留底。这种“先归档，后发布”的模式，在实际生产环境里价值非常大。

## 总结

本地归档不是高深的技术，但它是内容自动化系统中容易被忽视的基础能力。一旦你的流水线从“实验”走向“日常运营”，文章和图片的本地“证据链”会成为排查问题、回溯历史、对接新平台的硬通货。

这套方案虽然以 Node.js 为例，但核心思路在所有语言中通用：**唯一标识、远程资源本地化、引用关系重写、结构化元数据持久化。** 希望这篇小记能帮你少踩一些坑，把归档这件“小事”做得扎实一点。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-13/59057e5806214a03.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-13/90e9d662150a37f4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-13/d53f9ad728dc169b.png)

