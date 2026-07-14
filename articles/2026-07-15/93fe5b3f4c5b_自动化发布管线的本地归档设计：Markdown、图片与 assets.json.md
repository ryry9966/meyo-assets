---
title: 自动化发布管线的本地归档设计：Markdown、图片与 assets.json
feedId: 29147
source: 综合讨论
publishedAt: 2026-07-15
---

## 背景

在 OpenClaw / MCP 生态里，很多人已经把“选题 → 内容生成 → 配图 → 多平台发布”做成了全自动管线。一个典型的链路是：Agent 从话题库抽取主题，调用大模型生成文章，再驱动图像模型产出封面与插图，最后通过各平台 API 推送。

这个过程中，容易忽略一个工程化问题：**本地归档**。没有归档，就缺少调试依据；图片链接失效后无从恢复；想复用某篇内容的变体，只能重新调用生成接口。于是我们设计了一套最小可用的本地归档方案，把文章、图片和元数据组织成结构化目录，并用 `assets.json` 记录资产关系。

## 问题拆解

1. **文章正文** 需要持久化为可编辑的 Markdown，并保留生成时的元数据（模型、温度、时间等）。
2. **图片** 需要从临时 URL 下载到本地，否则 CDN 清理后内容会丢失。
3. **资产关联** 需要通过一个清单文件，把文章、图片和发布状态绑定在一起，方便后续检索和重放。
4. **自动化集成** 归档步骤不能要求人工操作，必须能在 MCP 工具或 Agent 流程中无缝调用。

## 做法与步骤

### 1. 约定目录结构

我们采用以“内容 ID”为主键的扁平目录，每个内容一个文件夹，内部统一布局：

```
posts/
  20250317-geoip-anti-fraud/
    article.md
    assets.json
    images/
      cover.png
      diagram-01.png
```

内容 ID 的命名规范为 `{YYYYMMDD}-{slug}`，slug 从文章标题自动生成（英文、小写、连字符）。

### 2. 生成并保存文章 Markdown

Agent 生成文章后，不直接发布，而是先调用一个“归档工具”。这个工具负责：

- 用 YAML front matter 记录核心元数据（生成时间、模型名、原始 prompt、输出 token 数等）；
- 将正文存储为 `article.md`；
- 从正文中提取图片引用（`![](...)`），交给图片下载模块。

示例 front matter：

```yaml
---
id: 20250317-geoip-anti-fraud
title: "GeoIP 在反欺诈系统中的工程实践"
created: 2025-03-17T14:23:00Z
model: claude-opus-4
temperature: 0.7
topics: [geoip, fraud, data-pipeline]
platforms: [medium, devto, wechat]
---
```

### 3. 图片本地化

自动发帖管线通常会先调用 DALL·E、Stable Diffusion 或 Midjourney 生成图片，返回临时 URL。归档时需要将图片下载到 `images/` 目录。

关键处理：

- **文件名清洗**：用内容 ID + 描述性命名（如 `cover.png`、`traffic-architecture.png`），避免原始随机字符串。
- **相对路径替换**：在 `article.md` 中，把远程 URL 替换为 `./images/xxx.png`，保证 Markdown 本地可预览。
- **图片尺寸与格式记录**：写入 `assets.json`，方便后续按需压缩或转换。

如果用的是 MCP 图片生成服务，图片下载可以直接在 MCP 工具里完成；否则用 `curl` 或 Python `requests` 实现一个轻量下载器，增加重试和校验（Content-Type 必须是 image/*）。

### 4. 构建 assets.json

`assets.json` 是整个归档的索引，推荐结构：

```json
{
  "id": "20250317-geoip-anti-fraud",
  "title": "GeoIP 在反欺诈系统中的工程实践",
  "created": "2025-03-17T14:23:00Z",
  "platforms": {
    "medium": { "post_id": "abc123", "published": "2025-03-17T16:01:00Z", "status": "published" },
    "devto": { "status": "draft" }
  },
  "images": [
    {
      "path": "images/cover.png",
      "alt": "GeoIP data pipeline overview",
      "generated_by": "dall-e-3",
      "prompt": "A clean data pipeline diagram...",
      "width": 1792,
      "height": 1024
    }
  ],
  "article_file": "article.md"
}
```

每次发布到新平台后，Agent 调用归档工具更新 `assets.json` 中相应平台的状态，保持本地记录与外部一致。

### 5. 在 Agent 流程中集成

我们把归档能力封装为两个 MCP 工具：

- `save_post_to_archive(post_id, markdown, metadata, images[])`：创建目录，写入 `article.md` 和 `assets.json`，下载图片。
- `update_platform_status(post_id, platform, status, remote_id)`：更新 `assets.json` 中某个平台的发布信息。

在 OpenClaw 的 graph 定义中，生成内容后立即调用 `save_post_to_archive`，一次配置即可自动运行。

## 踩坑点

- **图片下载可能失败或非图片**：远程 URL 返回 HTML 或 404 时，需要捕获异常，记录错误而不中断整个归档流程。在 `assets.json` 中将对应图片标记为 `download_failed: true`。
- **并发归档**：如果多个 Agent 实例同时处理不同内容，使用内容 ID 作为文件夹名可以天然避免目录冲突，但 JSON 文件更新需要加锁。简单场景下，可以串行化归档调用。
- **中文字段与编码**：`assets.json` 中的 `title` 可能包含中文，保存时要确保使用 UTF-8 编码，避免在 Windows 终端下出现乱码。
- **Markdown 预览路径**：在本地 Typora 或 VS Code 预览时，相对路径 `./images/` 没问题，但如果 `article.md` 被移动到其他位置，路径会断。我们通过约定“永远将 `article.md` 放在内容根目录”来规避。
- **文件系统权限**：在容器化环境中运行时，确保归档目录可写，或者使用挂载卷。不要向镜像层写入永久数据。

## 可复用建议

- **独立脚本化**：即使不依赖 MCP，也可以把归档逻辑写成一个独立 Python 脚本，通过命令行接收参数，方便在 GitHub Actions、Jenkins 等 CI 环境中直接调用。
- **扩展为知识库**：有了结构化的 `assets.json`，可以快速提取“过去 30 天生成的所有封面图的 prompt”，用于复盘生成效果。
- **与静态博客联动**：归档后的文件夹其实就是 Hugo、Hexo 等内容源，稍加转换就能生成一个本地知识库网站。
- **注意版权与容量**：图片本地化会占用磁盘空间，建议设置保留策略，比如超过 90 天未发布的文章可单独打包压缩。

## 总结

本地归档并不是“做完再补”，而是自动化发布管线必不可少的一层。通过约定清晰的目录结构、用 Markdown + front matter 保存文章、用 `assets.json` 管理资产与发布状态，并以 MCP 工具的形式集成进 Agent 工作流，能让每一次生成都有据可查、可重放、可审计。对于长期维护内容矩阵的个人或团队来说，这套轻量方案成本极低，却能大幅减少“内容丢失、图片失效、状态混乱”等问题。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-15/09c16798aaca1cf0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-15/8ad6ee6b387b5eeb.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-15/c72523a7372fa845.png)

