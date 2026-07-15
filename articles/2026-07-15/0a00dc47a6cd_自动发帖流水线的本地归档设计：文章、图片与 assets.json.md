---
title: 自动发帖流水线的本地归档设计：文章、图片与 assets.json
feedId: 29199
source: 综合讨论
publishedAt: 2026-07-15
---

## 背景

在 OpenClaw 或基于 Agent 的自动化实践中，不少同学会把“内容生成→多平台发布”串成一条流水线。比如利用定时触发器，每天让 LLM 生成一篇技术帖，再通过小红书、知乎、公众号等平台的 API 或模拟操作自动发布。这类流水线跑起来之后，一个容易被忽视的问题是：**生成的内容和图片散落在临时目录或平台草稿箱里，一旦想追溯、复用或本地重构，几乎没有结构化的归档方式。**

本篇文章梳理了一套轻量级的本地归档方案，将每次生成的**文章、图片**以及它们的**元数据**统一管理，并用一个 `assets.json` 作为索引，便于后续查询或重新发布。

## 核心问题

自动发帖流水线通常由这几个步骤串联：选题 → 文本生成 → 封面/配图生成 → 多平台分发。如果缺少归档环节，会遇到三个痛点：

1. **内容不可追溯**：几天前的帖子改了哪些内容、用了哪张图，全靠记忆或平台历史记录，平台一删就彻底丢失。
2. **图片与文章关系松散**：图片可能以随机 UUID 命名，散落在不同文件夹，很难人工匹配。
3. **跨平台发布不一致**：不同平台对尺寸、格式要求不同，没有统一资产映射，重发时需要重新处理。

解决办法是在流水线末尾增加一个**归档步骤**，把原始 Markdown、处理后的图片、平台差异化素材与一份结构化索引一并存入本地仓库。

## 做法与步骤

### 1. 约定本地归档目录结构

推荐以“一帖一目录”为原则，扁平化管理。例如：

```
archives/
├── 2025-03-12_gpu-profiling/
│   ├── article.md          # 原始 Markdown 正文
│   ├── cover.png           # 通用封面图
│   ├── carousel/           # 多图图文（如小红书轮播图）
│   │   ├── 01.png
│   │   └── 02.png
│   └── platform/           # 按平台裁剪的衍生资源
│       ├── zhihu_cover.jpg
│       └── xiaohongshu_carousel/
└── assets.json             # 总索引文件
```

`assets.json` 是一个 JSON 数组，每条记录描述一次内容资产的元信息。示例结构：

```json
[
  {
    "id": "2025-03-12_gpu-profiling",
    "title": "CUDA Graph 踩坑笔记",
    "created_at": "2025-03-12T10:00:00Z",
    "tags": ["cuda", "profiling", "debug"],
    "article": "2025-03-12_gpu-profiling/article.md",
    "cover": "2025-03-12_gpu-profiling/cover.png",
    "carousel": [
      "2025-03-12_gpu-profiling/carousel/01.png",
      "2025-03-12_gpu-profiling/carousel/02.png"
    ],
    "platform_assets": {
      "zhihu": {
        "cover": "2025-03-12_gpu-profiling/platform/zhihu_cover.jpg"
      },
      "xiaohongshu": {
        "carousel": "2025-03-12_gpu-profiling/platform/xiaohongshu_carousel/"
      }
    },
    "publish_records": [
      {"platform": "xiaohongshu", "url": "...", "published_at": "..."}
    ]
  }
]
```

### 2. 在发布流程中插入归档逻辑

以 OpenClaw 工作流为例，假设我们有一个 `publishPost` 的 Action，可以在当中调用一个 `archiveAssets` 的自定义 Node。关键步骤：

- **Article 保存**：将 LLM 生成的 Markdown 文本直接写入 `article.md`，不要进行平台适配的二次改写，保留最原始的正文。
- **图片保存**：如果使用 Stable Diffusion 等工具生成配图，生成后立即复制到对应目录；如果使用 Pillow 等库合成知识卡，则将合成结果写入，并保留合成参数（字体、字号、颜色）在元数据中，方便后续批量修改。
- **构建 assets.json**：先在内存中维护一个 `assetEntry` 对象，填充 id、标题、标签等，最后调用 `updateAssetsJson(entry)` 函数，读取已有的 `assets.json`，追加新条目或更新已有条目，再写回。

伪代码片段：

```python
def archive_post(post: dict, workspace: Path):
    slug = f"{post['date']}_{slugify(post['title'])}"
    post_dir = workspace / slug
    post_dir.mkdir(parents=True, exist_ok=True)
    
    # 保存文章
    (post_dir / "article.md").write_text(post["markdown"], encoding="utf-8")
    
    # 处理图片
    cover_path = post_dir / "cover.png"
    shutil.copy(post["cover_image"], cover_path)
    
    # 更新 assets.json
    assets = load_assets_json(workspace)
    entry = {
        "id": slug,
        "title": post["title"],
        "created_at": post["date"],
        "tags": post.get("tags", []),
        "article": f"{slug}/article.md",
        "cover": f"{slug}/cover.png",
        # ...
    }
    assets.append(entry)
    save_assets_json(workspace, assets)
```

### 3. 集成到 MCP 工具链

若你的流水线基于 MCP，可将归档功能封装成一个 MCP Server 的工具（如 `archive_post`），接受 id、title、article_path、cover_path 等参数。这样任何 Agent 在完成内容生成后，都可以通过调用该工具完成归档，无需重复实现磁盘写逻辑。

## 踩坑点

1. **图片命名与引用路径**  
   建议图片文件名全部使用英文 + 序号，避免中文字符在不同操作系统下出现路径问题。Markdown 内的图片引用在归档时最好替换为相对路径 `./cover.png`，这样即使整个 `archives/` 目录迁移，文章仍能正确显示。

2. **assets.json 的并发写入**  
   如果同一时间有多条流水线在归档，可能会出现两条记录交错写入导致文件损坏。简单的解决方案是使用文件锁（如 Python 的 `fasteners` 或直接 flock），或者改用 SQLite 存储元数据，避免直接操作 JSON。

3. **平台资产的尺寸与格式还原**  
   比如小红书的轮播图要求 3:4 竖版，知乎封面是 16:9。归档时不仅要保留最终图片，还要记录转换参数，否则下次想直接基于归档重发，仍需重新裁剪。

4. **索引文件日渐臃肿**  
   当帖子数积累到上千篇时，单个 `assets.json` 解析缓慢。可以用日期或标签做分片，比如 `assets-2025-Q1.json`，同时保留一个 `index.json` 指向各分片。

## 可复用建议

- **轻量化起步**：一开始不必设计复杂的数据库，一个 Markdown + JSON 的结构足矣，后续可以迁移。
- **与发布平台解耦**：归档只关心**内容本身**和**平台差异化素材**，不要包含平台特有的 session、cookie 等状态。
- **写单元测试**：归档逻辑的纯函数性质使其很容易测试——输入一份模拟的帖子 dict，检查输出文件夹和 JSON 内容。
- **作为 OpenClaw 的插件提供**：可将该归档模块封装成一个可选插件，提供“归档到本地”“归档到 GitHub 仓库”两种模式，方便社区复用。

## 总结

自动发帖流水线引入本地归档，实质上是为所有生成内容建立一份**不可变、可迁移的单一事实源**。这份事实源不依赖任何云平台，能随时被其他 Agent 或工具读取，进一步扩展出“历史内容润色重发”“多语言翻译复用原图”等场景。通过结构化 `assets.json`，你得到的不仅是一堆文件，还有一个可查询的内容资产清单。在实际工程中，先确保文章和图片落盘，再逐步完善画像元数据，是最稳妥的落地路径。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-15/84f17bf14d387afc.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-15/ef8a36c160f80092.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-15/8b5812e42c2c6c5c.png)

