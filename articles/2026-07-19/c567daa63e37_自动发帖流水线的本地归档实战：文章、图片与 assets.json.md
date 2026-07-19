---
title: 自动发帖流水线的本地归档实战：文章、图片与 assets.json
feedId: 29627
source: 综合讨论
publishedAt: 2026-07-19
---

## 背景

越来越多团队开始用 Agent 或管线脚本驱动内容发布：AI 生成 Markdown 正文、DALL·E/Stable Diffusion 出图，最后通过平台 API 一键推送。这套流水线解决了“从灵感到线上”的时效问题，却在工程上留了一个明显的缺口——**本地归档**。

不是没有意识到归档重要，而是大多实现把归档视为后顾之忧，结果线上内容更新几轮后，本地只剩一堆散落的 `.md` 和意义不明的 `img_001.png`，甚至连哪篇文章配了哪张图都搞不清楚。时间一长，想要重构旧文、复盘 prompt 效果，或者把优质内容迁移到自己的知识库，成本成倍翻涨。

## 问题拆解

我们需要的是一个轻量、机器可读、人也能一眼看懂的本地归档方案，满足三点基本要求：

1. **资产可追溯**：每篇文章的所有产出（`.md`、封面、内嵌图片）放在同一目录，名字有意义。
2. **元数据可查询**：用一份结构化的索引文件记录文章标题、slug、标签、生成时间、图片清单等，方便脚本或 Agent 后续复用。
3. **流程无侵入**：归档步骤应该作为流水线的最后一环自动完成，不增加手动操作，不需要打开额外工具。

如果缺少这些，流水线就只是一个“发后即焚”的管道，和内容运营需要的长期积累背道而驰。

## 做法与步骤

### 1. 目录结构约定

以文章唯一标识 `slug` 为基准，按日期分层。例如：

```
archive/
├── assets.json
├── 2025-04-28-how-i-automate-weekly-newsletter/
│   ├── article.md
│   ├── cover.png
│   ├── diagram-flow.png
│   └── meta.yaml          # 可选，原始生成参数
└── 2025-05-01-building-a-mcp-tool-for-archival/
    ├── article.md
    └── cover.png
```

`slug` 取自标题的 URL 安全化，结合日期前缀，天然避免重名。每篇文章一个独立目录，所有资产内部相对引用，外部工具只要拿到 `assets.json` 就能定位任何一篇内容。

### 2. 在流水线中插入归档步骤

假设你的流水线主要用 Python 或 Node.js 编排，在“生成内容 → 平台发布”之后，追加一个归档脚本 `archive_post.py`。伪逻辑：

```python
import frontmatter
import shutil, json, os
from datetime import date

def archive(article_md_path, image_paths, slug, metadata={}):
    today = date.today().isoformat()
    dir_name = f"{today}-{slug}"
    dest_dir = os.path.join("archive", dir_name)
    os.makedirs(dest_dir, exist_ok=True)

    # 1. 拷贝 markdown
    shutil.copy2(article_md_path, os.path.join(dest_dir, "article.md"))

    # 2. 拷贝所有图片（保留原名）
    for img in image_paths:
        shutil.copy2(img, dest_dir)

    # 3. 读取 frontmatter 中的标题、标签等
    with open(article_md_path) as f:
        post = frontmatter.load(f)
    meta = {
        "title": post.get("title"),
        "slug": slug,
        "date": today,
        "tags": post.get("tags", []),
        "images": [os.path.basename(p) for p in image_paths],
        **metadata
    }

    # 4. 更新根目录的 assets.json
    assets_path = "archive/assets.json"
    if os.path.exists(assets_path):
        with open(assets_path) as f:
            assets = json.load(f)
    else:
        assets = {}
    assets[slug] = meta
    with open(assets_path, "w") as f:
        json.dump(assets, f, indent=2, ensure_ascii=False)
```

对于 OpenClaw 用户，你可以把这段逻辑封装成一个 MCP 工具或本地插件，让 Agent 在完成发文后直接调用 `archive_post`，甚至把 `assets.json` 也作为上下文反馈给 Agent，使其能“记住”自己写过什么。

### 3. 与版本控制结合（推荐）

将 `archive/` 初始化为一个独立的 Git 仓库，在每次归档后自动提交：

```bash
cd archive
git add .
git commit -m "archive: ${slug}"
```

这样每次生成都留下快照，你可以回溯任何一版 prompt 和结果，也算是给 AI 产物加了一道人工或自动审计的保险。

## 踩坑点

- **图片引用路径问题**：Markdown 中引用图片时务必使用相对路径（如 `cover.png`），而不是绝对路径或本地临时路径。归档脚本可以在拷贝前替换一次路径，也可以养成从一开始就用相对路径的规范。  
- **assets.json 并发写入**：如果多条流水线可能同时运行，简单的读-改-写 JSON 会有竞争风险。对于单人使用或低频自动化问题不大，但量级上来后建议用小文件数据库（如 TinyDB）或按文章单独存 meta 文件，最后聚合生成 assets.json 作为只读索引。  
- **幂等性缺失**：重复归档同一 slug 可能覆盖已有文件。建议根据 slug 存在性判断是否使用 `exist_ok` 并记录日志，或者采用版本后缀 `v1, v2`。不想丢失历史就保留所有版本，并让 assets.json 中记录最新条目。  
- **流水线中途失败时的脏数据**：归档步骤应放在发布成功之后，如果前置步骤失败就不要写入档案。同时脚本本身需要做好异常捕获，避免留下不完整的目录。

## 可复用建议

1. **抽象成独立 CLI**：把上面的 `archive_post.py` 做成带参数的命令行工具，比如 `archive-cli --md article.md --img cover.png diagram.png --slug my-post`，方便任何流水线调用。  
2. **作为 MCP 服务提供**：如果你已经在用 OpenClaw 的 MCP 宿主，把归档功能注册为一个 `tool/archive`，Agent 在完成发文后即可自动触发，真正做到无人值守。  
3. **元数据尽早在生成阶段注入**：在 AI 生成内容时，就让模型输出标准的 frontmatter（标题、标签、描述等），这样归档脚本几乎零额外配置。  
4. **定期校验**：写一个简单的健康检查脚本，遍历 assets.json 中的条目，确认对应目录和图片都存在，保证索引和实际文件一致。

## 总结

本地归档看起来是“顺便做一下”的小工程，但对长期运营内容的团队来说，它是让流水线从一次性工具变成可积累资产的核心环节。文章、图片加上一份清晰的 `assets.json`，不仅让回溯变得简单，也为后续的检索、再发布、甚至训练私有数据集提供了干净的结构化输入。花一个小时把归档逻辑嵌入到现有流水线中，远比每隔几个月手动整理散落文件划算得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/2d6930bf0e22eaf3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/8a318fda42b390d6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/a6197481f79bdfc3.png)

