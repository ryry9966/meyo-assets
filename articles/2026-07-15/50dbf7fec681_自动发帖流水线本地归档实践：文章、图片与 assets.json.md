---
title: 自动发帖流水线本地归档实践：文章、图片与 assets.json
feedId: 29158
source: 综合讨论
publishedAt: 2026-07-15
---

## 背景

在基于 OpenClaw 搭建的自动化内容生产管线中，Agent 会按计划生成 Markdown 文章，并通过 MCP 调用图像生成服务（如 DALL·E、Stable Diffusion）为文章配图。这些产出物最终被推送到不同平台，例如博客、社交媒体、内部知识库。

早期我们只关心“发出去”这个动作，直到某天需要对历史内容做一次性审计，才发现流水线本身几乎没有留下任何可追溯的本地记录——部分图片链接已经过期，文章版本无法回退，生成时的元信息（模型、提示词、配置）也早已丢失。于是我们开始为这条流水线设计一个简单、可控的本地归档机制，将每次输出的文章、图片以及一份结构化的 `assets.json` 同步落盘。

## 问题拆解

归档的目标不是简单地复制文件，而是要让每条内容在未来能被独立重建、检查或复训。实际需要解决的问题包括：

1. **文章和图片的对应关系**：Markdown 中引用的图片可能是远程 URL，需要在归档时下载到本地，并修正引用路径。
2. **元信息的持久化**：发布平台、时间戳、使用的 prompt、模型版本、分类标签等，这些都不适合塞进文件属性，需要结构化记录。
3. **文件命名与目录组织**：高频率产出下，避免名称冲突，同时让归档结构可读、可检索。
4. **与现有流水线的集成成本**：不能引入重量级存储服务，也不能让归档操作拖慢发布节奏。

## 做法与步骤

我们的流水线核心是一个 OpenClaw 自定义脚本，在 `post_publish` 钩子中触发归档逻辑。整体流程如下：

### 1. 约定归档目录结构

```
archive/
└── 2025/
    └── 03/
        └── 21/
            ├── a3f2b1c4-202503211430/
            │   ├── post.md
            │   ├── images/
            │   │   ├── cover.png
            │   │   └── img_01.png
            │   └── assets.json
```

每一篇文章按 `date/uuid` 组织。UUID 在生成文章时便写入上下文，保证全链路可追踪。日期前置是为了避免单目录文件数膨胀，同时方便按时间范围快速定位。

### 2. 下载并重映射图片

Agent 生成的文章中图片通常是远程 URL。归档脚本执行时，会：

- 解析 Markdown 中的所有 `![alt](url)`；
- 检查 URL 是否可访问，对于已失效的临时链接（如 OpenAI 返回的临时 URL）会尝试通过原始生成服务重新获取或标记为缺失；
- 下载图片到本地 `images/`，统一重命名为 `cover.png`、`img_01.png` 等，保持引用稳定；
- 将 Markdown 中的图片链接替换为相对路径 `./images/cover.png`，使归档后的文章可以脱离网络阅读。

```python
for i, url in enumerate(image_urls):
    local_name = "cover.png" if i == 0 else f"img_{i:02d}.png"
    download_image(url, archive_dir / "images" / local_name)
    # rewrite md content
```

### 3. 生成 assets.json

`assets.json` 是归档的核心元数据文件，记录该次产出的一切关键信息，典型字段包括：

```json
{
  "id": "a3f2b1c4...",
  "title": "How We Archive Post Pipelines",
  "created_at": "2025-03-21T14:30:00Z",
  "platform": "openclaw-cn",
  "tags": ["automation", "archiving"],
  "model": "gpt-4o",
  "image_model": "dall-e-3",
  "prompts": {
    "text": "...",
    "images": ["cover prompt", "img1 prompt"]
  },
  "files": {
    "md": "post.md",
    "images": ["images/cover.png", "images/img_01.png"]
  }
}
```

这份 JSON 不仅方便人或程序检索，也直接作为未来重训练 or 数据清洗的结构化输入。

### 4. 集成到 OpenClaw 流水线

通过 OpenClaw 的 plugin / hook 机制，在内容发布成功之后异步执行归档函数，不阻塞主流程。示例如下（伪代码）：

```python
@hook("post_publish")
def archive_artifacts(post, context):
    archive_path = generate_archive_path(post.id, post.created_at)
    save_md(archive_path / "post.md", post.content)
    images = download_and_remap(post.image_urls, archive_path)
    assets = build_assets(post, images, context)
    write_json(archive_path / "assets.json", assets)
```

对于没有内置 hook 系统的情况，也可以用文件系统监视器或定时任务扫描待归档目录来解耦。

## 踩坑与排障

实际使用中踩了几个典型的坑：

- **临时图片链接过期**  
  某些图像生成服务返回的 URL 仅在数小时内有效。等到归档脚本运行时，图片已无法下载。解决方案是将生成后的图片立即写入临时本地缓存，或者请求服务返回 base64 数据，由流水线直接处理为本地文件。
- **并发写入冲突**  
  如果多条流水线同时触发归档，目录创建和文件写入可能产生竞争。使用 UUID 命名加上 `mkdir(exist_ok=True)` 的原子操作可以规避，但在极高并发下最好引入一个轻量的文件锁。
- **相对路径与绝对路径的混淆**  
  Markdown 内嵌图片路径在归档后应当总是相对路径，否则离开原始环境就无法打开。自动替换时需注意不要改动外部链接（如 `http://` 开头的引用），可以仅替换已知的图片域名或标记过的链接。
- **JSON 序列化中的特殊字符**  
  文章的 prompt 可能包含多行长文本、特殊符号，直接序列化可能破坏 JSON 结构。始终使用标准 JSON 库并开启 ensure_ascii=False，或将超长字段先做清理。

## 可复用建议

如果你的流水线也需要本地归档，以下几点可直接复用：

- **尽早生成唯一 ID**：文章一经创建就赋予 UUID，贯穿日志、文件名和 metadata。
- **目录分层**：按 `yyyy/mm/dd/` 分层，兼顾可读性和性能。
- **图片本地化永远不晚**：不要依赖外部 URL 的持久性，下载到本地是最稳妥的一步。
- **归档与发布解耦**：归档逻辑应异步、非阻塞，失败时不影响正常发布，但要记录错误日志。
- **考虑版本化和清理策略**：可以将归档目录纳入 Git（配合 `.gitignore` 忽略大文件或使用 Git LFS），同时设置 TTL 清理规则，防止磁盘占满。
- **assets.json 的 schema 演进**：用版本号标记 JSON 结构，`"schema_version": "1.0"`，未来解析时能兼容老数据。

## 总结

一个可靠的本地归档机制，本质上是在为自动化内容生产管线构建“数据账本”。文章、图片与 assets.json 三者配合，让每一条自动产出的内容都变得可回溯、可复用、可审计。在 OpenClaw 的生态里，这不需要引入额外的基础设施，几十行 Python 和一个稳定的目录约定就能实现。持续产出内容很重要，但为这些内容保留一份完整的本地副本，是我们对工程可控性最低限度的尊重。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-15/26942864176dd56b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-15/21365799c4ea369c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-15/87e999b1540348c9.png)

