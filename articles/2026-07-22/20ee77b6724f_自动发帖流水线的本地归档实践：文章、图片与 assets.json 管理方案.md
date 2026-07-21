---
title: 自动发帖流水线的本地归档实践：文章、图片与 assets.json 管理方案
feedId: 29986
source: 综合讨论
publishedAt: 2026-07-22
---

## 背景

用 OpenClaw 或 Agent 搭建自动发帖流水线之后，内容产出的速度会明显提升。但当文章、配图、中间产物都以散落文件的形式堆积在工作目录里，用不了多久就会变得难以维护。更麻烦的是，一旦远端平台出现限流、封号或者服务故障，历史内容就会面临不可追溯的风险。

我们在一套基于 MCP + 定时 Agent 的发帖流水线中遇到了类似的痛点：每次运行会生成 Markdown 正文、一张封面图以及若干内嵌图片，但这些文件缺少统一的索引，后续想复用、改稿或者做内容审计时，只能靠人工翻目录。

因此，我们设计了一套轻量级的本地归档规范：所有输出按日期分桶，文章、图片以及一个 `assets.json` 索引文件共同构成一次完整的发布记录。整个方案不依赖外部数据库，只依赖文件系统与少量脚本，可嵌入到现有的 Agent 工具链中。

## 问题拆解

流水线中需要解决三个核心问题：

1. **文件组织**：同一批次产出的文章和图片必须能快速定位，且不与其他批次冲突。
2. **索引与追溯**：需要知道某篇文章对应哪些图片、生成时间、发布目标平台、关键元数据，方便后续查询。
3. **工程化可靠性**：归档过程不能因为文件名重复、并发写入或路径问题导致数据丢失或覆盖。

如果只是简单地把文件扔进 `output/` 目录，用日期前缀命名，很快就会出现“图片和文章对应不上”“同一文章改了多版无法分辨”之类的问题。因此需要一个结构化的约定，并通过一个 JSON 文件作为每批产出的自描述索引。

## 归档结构设计

最终落地的目录结构如下：

```
archive/
└── 2025-01-20/
    ├── article.md
    ├── cover.png
    ├── img_01.png
    ├── img_02.png
    └── assets.json
```

- **日期目录**：以 `YYYY-MM-DD` 命名，同一天多次运行会复用同一目录（后面会讲冲突处理）。
- **article.md**：完整的 Markdown 正文，图片引用已改写为相对路径 `./cover.png`、`./img_01.png`，确保在本地 Markdown 编辑器里可以直接预览。
- **图片文件**：封面统一命名为 `cover.<ext>`，其他按序号命名。如果一天内有多篇文章产出，则在文件名中加入短哈希或时间戳以避免覆盖（如 `article_a3f2.md`）。
- **assets.json**：记录本次产出的所有文件清单及元数据。一个典型的 `assets.json` 结构如下：

```json
{
  "uid": "2025-01-20-14-30-15-a3f2",
  "created_at": "2025-01-20T14:30:15Z",
  "platform": "wechat",
  "title": "自动化流水线归档实践",
  "files": {
    "article": "article_a3f2.md",
    "cover": "cover_a3f2.png",
    "images": ["img_01_a3f2.png", "img_02_a3f2.png"]
  },
  "meta": {
    "word_count": 950,
    "image_model": "dalle3",
    "prompt_hash": "e4d9f..."
  }
}
```

`uid` 是我们生成的唯一标识，由日期、时间、短随机串组成，既容易按时间排序，又能避免碰撞。所有文件都通过 `uid` 前缀关联，而不是仅靠所在目录，这样即使文件被意外移动，通过 `assets.json` 也能重建关联。

## 流水线中的实现步骤

以 OpenClaw 的 Agent 工作流为例，我们拆分出三个关键节点：

### 1. 内容生成与文件写入

Agent 先调用 LLM 生成 Markdown 正文，同时通过 MCP 图像生成服务产出封面和配图。关键点是：**生成阶段先用内存或临时目录承载，不要直接写入归档目录**。

临时目录中会得到类似这样的文件：
```
/tmp/post_work/article.md
/tmp/post_work/cover.png
/tmp/post_work/img_01.png
/tmp/post_work/img_02.png
```

然后由归档脚本统一处理，避免半成品被误归档。

### 2. 归档脚本（Python 示例核心逻辑）

```python
import os, json, shutil, uuid, hashlib
from datetime import datetime

def archive_post(temp_dir, archive_root, platform, title):
    now = datetime.utcnow()
    date_str = now.strftime("%Y-%m-%d")
    uid = now.strftime("%Y-%m-%d-%H-%M-%S") + "-" + uuid.uuid4().hex[:4]
    dest_dir = os.path.join(archive_root, date_str)
    os.makedirs(dest_dir, exist_ok=True)

    # 重命名并复制文件
    article_src = os.path.join(temp_dir, "article.md")
    article_dst = os.path.join(dest_dir, f"article_{uid}.md")
    shutil.copy2(article_src, article_dst)

    cover_src = os.path.join(temp_dir, "cover.png")
    cover_dst = os.path.join(dest_dir, f"cover_{uid}.png")
    shutil.copy2(cover_src, cover_dst)

    img_files = []
    for i, fname in enumerate(sorted(os.listdir(temp_dir))):
        if fname.startswith("img_") and fname.endswith(".png"):
            dst = os.path.join(dest_dir, f"img_{i+1:02d}_{uid}.png")
            shutil.copy2(os.path.join(temp_dir, fname), dst)
            img_files.append(os.path.basename(dst))

    # 写入 assets.json
    assets = {
        "uid": uid,
        "created_at": now.isoformat() + "Z",
        "platform": platform,
        "title": title,
        "files": {
            "article": os.path.basename(article_dst),
            "cover": os.path.basename(cover_dst),
            "images": img_files
        },
        "meta": {
            "word_count": len(open(article_dst).read().split()),
            "image_model": "dalle3",
            "prompt_hash": hashlib.md5(open(cover_src, "rb").read()).hexdigest()[:6]
        }
    }
    assets_path = os.path.join(dest_dir, f"assets_{uid}.json")
    with open(assets_path, "w") as f:
        json.dump(assets, f, indent=2, ensure_ascii=False)

    return assets_path
```

这段脚本会处理文件重命名、移动以及 `assets.json` 生成。为了支持一天内多次归档，每个 `assets.json` 也是独立的（带 uid），这样就不会出现覆盖问题。如果想快速查询某一天所有产出，只需 `ls archive/2025-01-20/assets_*.json` 然后合并读取即可。

### 3. Agent 节点的集成

在 OpenClaw 的工作流配置中，我们将上面的归档脚本封装成一个 Tool 或 MCP 资源，供 Agent 在完成发帖动作后调用。大致流程：

1. 内容 Agent 执行生成任务，输出临时目录路径与元信息。
2. 调用 `local-archive` Tool，传入临时目录、归档根目录、平台名称、标题。
3. Tool 执行归档脚本，返回 `assets.json` 的路径。
4. 后续节点（比如通知、备份）可以读取这个路径来获取所有文件信息。

## 踩坑记录

- **路径冲突问题**：最早我们直接用 `article.md` 不带 uid，结果同一天两次运行直接覆盖了上一篇文章。改用 `uid` 前缀彻底解决，但也需要约定 `assets.json` 的文件名同样带上 `uid`，否则自己也互相覆盖。
- **图片格式不一致**：不同图像生成服务可能返回 JPEG、PNG 甚至 WebP，直接硬编码扩展名 `.png` 会导致文件打不开。我们在归档脚本中增加了格式检测，通过文件头魔数决定扩展名，或者统一用 `cover.<实际扩展名>` 并记录在 `assets.json` 中。
- **相对路径改写**：生成的 Markdown 中图片链接可能是绝对 URL（指向临时图床），如果直接归档，本地打开就看不到图。我们在归档前会通过正则将所有图片链接替换为相对路径 `./cover_uid.png` 等，确保本地可读。
- **并发安全问题**：如果短时间内触发多次归档，且都在同一日期目录下操作，虽然 `uid` 不同不会覆盖，但高并发下仍可能出现临时文件残留。建议归档脚本内部对同一日期目录加文件锁（如 `fcntl`）或者将归档动作串行化。

## 可复用建议

1. **目录结构的标准化**：无论使用什么 Agent 框架，只要遵循 `archive/YYYY-MM-DD/assets_uid.json` 的模式，就能用简单的脚本快速索引与查询。
2. **元数据尽量自描述**：在 `assets.json` 中记录文章字数、使用的模型、图片 prompt 摘要，可以极大方便后期筛选和打标签，无需重新解析文章全文。
3. **定期备份与清理**：归档目录本身可以再通过 `rsync` 或对象存储同步到远端，本地只保留最近 30 天；或者设置定时任务将超过 90 天的内容压缩打包，降低索引压力。
4. **与外部工具的衔接**：通过 `assets.json`，可以轻松写出检索工具（比如「列出上个月所有包含特定关键词的文章」），也可以将归档目录整体挂载到笔记软件中作为知识库。

## 总结

本地归档并不是一个花哨的需求，却是自动化内容流水线能够长期稳定运营的基础建设。通过引入 `assets.json` 作为每次产出的自描述索引，配合带 uid 的文件命名与日期分桶，我们花较少的工程成本换来了完整的可追溯性、可复用性与灾难恢复能力。

对整个 OpenClaw / Agent 社区而言，这种小而确定的模式非常适合嵌入到各类自建发布管线中。不要等到文件堆积如山、图片丢失、文章找不到时才想起归档——在一开始就把它设计进流水线的最后一个环节。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/01132302fcc4cd3a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/8930d564427ddad5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/6327ebcc55206a16.png)

