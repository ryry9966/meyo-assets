---
title: 自动发帖流水线的本地归档实战：文章、图片与 assets.json
feedId: 29123
source: 综合讨论
publishedAt: 2026-07-15
---

## 背景

在基于 OpenClaw/Agent/MCP 的自动化内容生产链路里，一条常见的流水线是：Agent 生成文案 → 配图生图 → 格式化为 Markdown → 推送到目标平台。这个过程中，大部分精力都花在“发布”这一动作上，历史内容长期散落在各个平台的草稿箱、剪切板或临时目录里。一旦需要回溯某篇旧文、替换图片、或迁移至其他平台，就会发现缺少一份完整且可追溯的本地副本。

因此，在流水线中引入一个轻量级的本地归档步骤，对长期维护自动化内容体系很有帮助。本文整理了一套在本地以文件系统 + `assets.json` 进行归档的实践方案，并分享与 Agent 工具链集成时遇到的坑和可复用经验。

## 问题拆解

我们将本地归档的需求拆成三个部分：

1. **文章正文**：最终输出的 Markdown，图片引用需指向本地相对路径。
2. **图片资源**：正文中引用的所有图片，不管是 AI 生成后直接使用的图片，还是从外部下载的，统一落入指定目录。
3. **元数据**：记录文章标识、创建时间、标签、发布平台、图片清单等信息，便于外部工具索引。

目标是让任意一次生产流水线结束后，`posts/{date-slug}/` 目录下都能得到自包含的完整归档，后续通过 `cat index.md` 就能完全还原内容。

## 做法与步骤

### 1. 确定归档结构

每个归档单元对应一次完整产出，结构如下：

```
posts/
└── 20250318-my-post-slug/
    ├── index.md          # Markdown 正文，图片引用相对路径
    ├── images/           # 所有图片文件
    └── assets.json       # 元数据
```

`assets.json` 示例：

```json
{
  "slug": "my-post-slug",
  "title": "Post Title",
  "created_at": "2025-03-18T10:30:00Z",
  "tags": ["automation", "openclaw"],
  "platforms": ["zhihu", "weixin"],
  "images": [
    {
      "file": "images/cover.png",
      "alt": "cover",
      "source": "generated",
      "prompt": "a robot writing on paper, tech illustration"
    }
  ],
  "version": "1"
}
```

### 2. 在流水线中加入归档节点

我们可以将归档实现为一个独立脚本（Python/Node.js），并通过 MCP Server 或 OpenClaw 的自定义 action 暴露给 Agent。在生成完文章和图片后，按顺序执行：

- 接收 Agent 传入的文章内容、文章元数据、图片清单（本地路径或临时文件）。
- 创建归档目录 `posts/{date-slug}/`。
- 将 Markdown 中的图片引用路径统一替换为 `./images/xxx.png`。
- 将图片文件复制/移动到 `images/`，若是远程 URL 则先下载。
- 生成 `assets.json` 并写入。
- 返回归档路径，供后续发布步骤使用。

如果在 Agent 内部使用 Python，伪代码大致如下：

```python
def archive_post(md_content, slug, meta, image_map):
    root = Path("posts") / f"{today()}-{slug}"
    root.mkdir(parents=True, exist_ok=True)
    img_dir = root / "images"
    img_dir.mkdir(exist_ok=True)
    # 处理图片
    for orig_ref, local_name in image_map.items():
        if orig_ref.startswith("http"):
            download(orig_ref, img_dir / local_name)
        else:
            shutil.copy(orig_ref, img_dir / local_name)
        md_content = md_content.replace(orig_ref, f"./images/{local_name}")
    # 写入文章
    (root / "index.md").write_text(md_content, encoding="utf-8")
    # 生成 assets.json
    assets = {
        "slug": slug,
        "title": meta.get("title"),
        "created_at": datetime.now(timezone.utc).isoformat(),
        "tags": meta.get("tags", []),
        "platforms": meta.get("platforms", []),
        "images": [{"file": f"images/{n}", "alt": a, "source": s, "prompt": p} for ...],
        "version": "1"
    }
    (root / "assets.json").write_text(json.dumps(assets, ensure_ascii=False, indent=2))
    return str(root)
```

### 3. 集成到 OpenClaw 工作流

如果使用 OpenClaw 搭建 Agent 流水线，可以把归档脚本封装为一个 MCP 工具 `local_archive`，参数为 Markdown 字符串、slug、元数据、以及一个图片映射 JSON。在编排工作流时，让生成文章、生图的 Agent 结束后调用该工具。这样做的好处是归档操作解耦，不侵入生成逻辑，也方便单独测试。

## 踩坑记录

### 图片引用替换不彻底

Markdown 中有可能出现 `![alt](url)`、`<img src="url">` 等多种写法，简单的字符串替换容易遗漏。我们的做法是先用 Markdown 解析器提取所有图片节点，再统一替换，但解析器可能对自定义属性支持不好。折中方案是：约定生成环节只输出标准 `![alt](url)` 格式，并在归档前做一次检查，若发现其他格式则报错，由 Agent 重试修正。

### 图片下载失败导致归档中断

网络波动或图床临时不可用会导致归档流程失败。改进措施：为下载添加重试逻辑（3 次指数退避），若最终仍失败，则在 `assets.json` 中记录状态为 `missing`，并将图片引用保持为原始 URL 并追加 `<!-- TODO: missing image -->` 注释，不阻塞归档过程。

### 并发写入冲突

当多个 Agent 实例同时运行并生成相同 slug 时，可能会导致覆盖或写入混乱。解决方案：用文件锁（如 `portalocker`）在创建目录前加锁，并在 slug 中加入毫秒级时间戳或 UUID 短码以保证唯一性。权衡后，我们选择将 slug 生成放在归档脚本内部，确保目录名唯一，而不是完全依赖 Agent 传入的 slug。

### assets.json 膨胀与版本迭代

随着归档需求变化，assets.json 字段会不断增加。建议从一开始就带有 `version` 字段，并为解析脚本提供版本兼容逻辑。我们目前用 `version: "1"`，未来新增字段时只做追加，保持向后兼容。

## 可复用建议

- **工具化**：把归档功能做成可单独运行的 CLI 或 MCP 工具，不要混在生成逻辑中。这样可以直接与任何 Agent 框架（OpenClaw、LangChain 等）搭配使用。
- **图片处理模块化**：分离图片下载、格式转换、重命名、引用替换的职责，方便单独测试和替换不同图床。
- **元数据索引**：除了每个归档单元内的 `assets.json`，还可以维护一个全局索引 `posts/index.json`，列出所有归档的文章列表和摘要，便于做历史搜索。这个索引由归档脚本在每次操作后增量更新。
- **与发布状态联动**：在 `assets.json` 中记录发布平台的 ID 或链接，后续可以写一个小服务来比对本地归档与线上状态，发现不一致时报警。

## 总结

本地归档看似简单，但在自动化内容流水线中很容易被忽视。一个可靠的文件级归档系统能让你在后续扩展、迁移、故障恢复时节省大量时间。我们通过面向目录的结构化归档，配合 `assets.json` 元数据，将每次产出都固化为一组可溯源的本地文件。整个方案轻量、无外部依赖，很适合与 OpenClaw 生态中的 MCP 工具集成，补齐“生产-发布”链条中缺失的存储环节。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-15/88ef5a1ad7b18e2c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-15/583b719306439101.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-15/1c3fab0921c06662.png)

