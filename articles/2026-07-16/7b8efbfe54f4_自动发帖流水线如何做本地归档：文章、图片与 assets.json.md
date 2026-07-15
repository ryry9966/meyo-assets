---
title: 自动发帖流水线如何做本地归档：文章、图片与 assets.json
feedId: 29237
source: 综合讨论
publishedAt: 2026-07-16
---

## 背景

很多基于 OpenClaw、Agent 或 MCP 构建的内容发布流水线，已经能够自动生成文章并推送到多平台。但这些流水线通常只关注“发出去”这一动作，却忽略了内容本身的本地留存。

带来的问题很直接：平台上的图片某一天变成了 404，你手里只剩下一个链接失效的 Markdown 草稿；或者想复盘某天 Agent 到底产出了一篇什么样的内容，却找不到可离线查看的版本；更麻烦的是，当同一个内容要迁移到新平台时，你会发现需要重新组织资源，而原始图片资源早已散落各处。

所以，本地归档并不是多余的步骤，而是内容流水线工程化绕不开的一环。本文给出一套轻量级、易集成的归档方案，专门面向自动化流水线中文章、图片和资源清单的本地留存。

## 问题拆解

自动发帖流水线的输出通常包含：文章标题、正文 Markdown、正文中引用的图片 URL（可能是临时链接或平台 CDN 地址，甚至是 base64 inline 数据）。归档需要解决以下问题：

1. **离线可读**：所有图片必须本地化，正文中的引用替换为相对路径。
2. **资源可溯源**：知道本地图片和原始 URL 的对应关系，便于后续排查或重新上传。
3. **结构规整**：支持批量归档，方便后续被脚本或工具消费（例如生成静态站、导入知识库）。
4. **流水线友好**：能以最少的侵入方式嵌入现有 Agent 或 MCP 流程，不打断主链路。

## 做法与步骤

### 1. 确定目录结构

每个文章一个独立文件夹，包含以下内容：

```
post-20250412-001/
├── index.md          # 已替换图片路径的归档正文
├── images/
│   ├── a1b2c3.png
│   └── d4e5f6.jpg
└── assets.json       # 资源清单
```

文章标题和日期等信息可以用 slug 体现，也可以全放入 `assets.json`。

### 2. 标准化 assets.json 格式

`assets.json` 是一条结构化的记录，推荐至少包含：

```json
{
  "id": "20250412-001",
  "title": "...",
  "created": "2025-04-12T10:30:00Z",
  "platforms": ["twitter", "blog"],
  "images": [
    {
      "originalUrl": "https://cdn.example.com/img1.png",
      "localPath": "images/a1b2c3.png",
      "hash": "sha256:...",
      "downloadedAt": "2025-04-12T10:31:02Z"
    }
  ],
  "sourceMarkdown": "原始的未经替换的 Markdown 文本（可选）"
}
```

`images` 中记录每一次下载的资源，`hash` 可用于去重或完整性校验，`platforms` 标记该稿件的分发渠道，方便后续追溯。

### 3. 实现归档脚本

归档核心逻辑可以拆成一个独立函数或 MCP tool，输入为文章对象（标题、正文、图片 URL 列表），输出为归档文件夹。这里以 Node.js 伪代码为例：

```javascript
async function archivePost({ title, markdown, imageUrls, outputDir }) {
  const slug = slugify(title) + '-' + Date.now();
  const postDir = path.join(outputDir, slug);
  const imagesDir = path.join(postDir, 'images');
  fs.mkdirSync(imagesDir, { recursive: true });

  const assets = { id: slug, title, images: [] };
  let localMarkdown = markdown;

  for (const url of imageUrls) {
    const hash = sha256(url);
    const ext = path.extname(new URL(url).pathname) || '.png';
    const localName = `${hash}${ext}`;
    const localPath = path.join('images', localName);

    try {
      await downloadFile(url, path.join(postDir, localPath));
      assets.images.push({
        originalUrl: url,
        localPath,
        hash: `sha256:${hash}`,
        downloadedAt: new Date().toISOString()
      });
      // 替换正文中所有该 URL
      localMarkdown = localMarkdown.replaceAll(url, localPath);
    } catch (err) {
      console.error(`下载失败: ${url}`, err.message);
      // 保留原始 URL，但标记下载失败
      assets.images.push({ originalUrl: url, localPath: null, error: err.message });
    }
  }

  fs.writeFileSync(path.join(postDir, 'index.md'), localMarkdown);
  fs.writeFileSync(
    path.join(postDir, 'assets.json'),
    JSON.stringify(assets, null, 2)
  );
  return postDir;
}
```

`downloadFile` 需要处理超时、重试、状态码等，推荐使用 `node-fetch` 加上合理的 `User-Agent`，避免被反爬。

### 4. 集成到流水线

根据你的架构，可以有几种集成方式：

- **作为 MCP tool**：由 Agent 在生成内容后调用，将 Markdown 和图片 URL 列表传入，归档到本地 NAS 或挂载目录。
- **作为发布前的 pre-hook**：在推送到平台的脚本中，先执行归档步骤，确保发布前已留存。
- **独立守护进程**：监听流水线日志或消息队列，异步消费发布事件并完成归档，不阻塞主链路。

无论哪种方式，归档动作失败不应该阻断发布（除非归档是强需求），但需要记录日志并触发告警。

### 5. 验证归档质量

归档完成后，建议自动执行以下检查：

- 本地 `index.md` 用 Markdown 预览工具打开，图片是否正常渲染。
- `assets.json` 中的每一条 `localPath` 是否有效，`hash` 是否与文件内容一致。
- 统计下载失败项，超出阈值则告警。

这些检查可以做成一个独立的验证脚本，在流水线末尾运行。

## 踩坑记录

- **图片 URL 格式不统一**：有时图片链接可能来自内网、需要特殊 Cookie；有的平台返回的 URL 是转存过的，并非原始上传地址。正则匹配时注意不要漏掉不带扩展名的图片链接（如 `/img?id=123`），这类需要根据响应 `Content-Type` 确定扩展名。
- **反爬与限速**：对某些 CDN 连续下载会触发 403 或限速，需要加入随机延迟、IP 轮换，或提前将图片托管至自己的图床后再归档。
- **文件名冲突**：我踩过直接用 `Date.now()` 命名的坑，并发归档时会出现覆盖。现在统一用 URL 的 SHA-256 作为文件名，基本不会冲突。
- **大文件**：曾有文章引用了体积巨大的高清示意图，归档一张图耗时几十秒，后来加了 20MB 上限，超出的记录在 `assets.json` 中但只保存原始 URL，不强制本地下载。
- **Windows 路径问题**：`localPath` 统一使用正斜杠 `/`，避免跨平台乱码。

## 可复用建议

- 这套归档逻辑可以封装成一个 `archive-kit`，直接引入到不同的 MCP Server 中，节省重复开发。
- 归档后的文件夹结构天然支持被静态站点生成器（如 Hugo、VitePress）消费，可以作为内部知识库或内容备份站。
- 当需要跨平台重新发布时，只需读取 `assets.json` 和 `index.md`，就能还原完整素材，无需去各平台翻找。
- 建议定期对归档目录做增量备份或 git 提交，形成版本历史。

## 总结

本地归档听起来是一件小事，但它解决了自动化内容流水线里“断了的就再也找不回”的问题。通过定义清晰的目录结构、用 `assets.json` 记录资源映射、以及几步简单的脚本，就能让每一条自动产出的内容真正落地、可追溯、可复用。

如果你已经在用 Agent 或 MCP 构建内容发布流程，不妨花一个小时把归档环节补上——它会在图片失效、平台改版或者需要回溯分析时，让你感谢自己当初做的这个决定。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/af0656491da25a97.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/414db1b63c656525.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/823b0ee45c1fdad0.png)

