---
title: 自动发帖流水线的本地归档方案：文章、图片与 assets.json 实战
feedId: 29588
source: 综合讨论
publishedAt: 2026-07-19
---

## 背景

在基于 OpenClaw 多 Agent 系统构建内容自动发布流水线时，文章和配图往往由不同工具或 Agent 生成，随后推送到微信公众号、知乎、Twitter 等平台。整个流程中，最容易忽视但又最致命的缺口，就是“本地归档”——即在发布动作完成之前，把所有产物（markdown 原文、图片文件、元数据）按可追溯的结构存回本地或 Git 仓库。

没有归档意味着：
- 线上图片链接失效后，本地再也无法复现整篇帖子的完整形态；
- 需要复用某篇旧文素材时，找不到当时到底生成了哪些图；
- 多轮迭代后，Agent 的输出散落在日志里，难以进行版本对比和质量审计。

因此，我在这条流水线中强制插入了一个本地归档步骤，用标准化的目录结构和 `assets.json` 来终结这种混乱。这篇文章会把实现细节、踩过的坑以及可复用的思路都摊开来讲。

## 问题拆解

自动发帖流水线通常长这样：
1. 选题 Agent 确定主题；
2. 写作 Agent 输出 markdown 正文；
3. 配图 Agent 生成若干图片（返回 URL 或 base64）；
4. 发布 Agent 组装内容并推送到目标平台。

归档需要在步骤 3 和 4 之间完成，并回答三个问题：
- 文件放哪里、怎么命名？
- 如何把文章、图片和元数据准确地关联起来？
- 如何做到原子化写入，不因并发或中途崩溃产生脏数据？

## 做法与步骤

### 1. 归档目录结构

我选择的方案是以 **一个帖子一个目录** 为单位，目录名使用全局唯一的 `post_id`（比如基于时间戳的 `20250413-143021`）。内部结构如下：

```
posts/
└── 20250413-143021/
    ├── post.md
    ├── images/
    │   ├── cover.png
    │   └── diagram_01.png
    └── assets.json
```

- `post.md`：最终要发布的 markdown 正文，图片引用全部使用相对路径 `./images/xxx.png`。
- `images/`：存放所有配图，文件名统一规范，不保留原始散乱的 hash 名。
- `assets.json`：记录该帖子的完整元数据与产物清单。

### 2. assets.json 的设计

`assets.json` 不是简单的文件列表，而是让机器和人都能快速理解帖子内容与结构的一份“索引”。我目前使用的字段如下（可根据实际流水线扩展）：

```json
{
  "post_id": "20250413-143021",
  "title": "Example Post Title",
  "created_at": "2025-04-13T14:30:21Z",
  "topics": ["automation", "openclaw"],
  "summary": "本文介绍…",
  "word_count": 850,
  "files": {
    "markdown": "post.md",
    "images": [
      {
        "file": "images/cover.png",
        "alt": "封面图",
        "generated_by": "image_agent_v2",
        "prompt_hash": "abc123"
      },
      {
        "file": "images/diagram_01.png",
        "alt": "架构图",
        "generated_by": "diagram_agent",
        "prompt_hash": "def456"
      }
    ]
  },
  "publish_targets": ["wechat", "twitter"],
  "version": 1
}
```

几个细节值得注意：
- 每张图片都记录了生成来源和 prompt 的哈希值，方便后续追溯“同一份 prompt 在不同模型下的输出差异”。
- `publish_targets` 会记录该帖最终发布到了哪些平台，方便做跨平台一致性检查。
- `version` 字段为后续可能的增量更新预留空间。

### 3. 在流水线中插入归档动作

归档逻辑被封装成一个独立的函数（或 MCP Tool），在发布 Agent 之前调用。简化版伪代码：

```python
def archive_post(post_id, markdown_content, images_list):
    post_dir = Path("posts") / post_id
    post_dir.mkdir(parents=True, exist_ok=True)

    # 写入 markdown
    (post_dir / "post.md").write_text(markdown_content, encoding="utf-8")

    # 下载/移动图片到 images/
    img_dir = post_dir / "images"
    img_dir.mkdir(exist_ok=True)
    image_records = []
    for idx, img in enumerate(images_list):
        target_name = img.get("target_name") or f"img_{idx:03d}.png"
        if img["type"] == "url":
            download_image(img["url"], img_dir / target_name)
        elif img["type"] == "base64":
            decode_base64_to_file(img["data"], img_dir / target_name)
        image_records.append({
            "file": f"images/{target_name}",
            "alt": img.get("alt", ""),
            "generated_by": img.get("generated_by", ""),
            "prompt_hash": img.get("prompt_hash", "")
        })

    # 构建并写入 assets.json
    assets = {
        "post_id": post_id,
        "title": extract_title(markdown_content),
        ...
        "files": {"markdown": "post.md", "images": image_records}
    }
    atomic_write_json(post_dir / "assets.json", assets)
```

如果你在用 OpenClaw，可以把这段逻辑挂载到一个自定义 Action 上，或者通过 MCP Server 暴露一个 `archive_post` 工具，让下游 Agent 直接调用，既解耦又方便复用。

## 踩坑点

1. **图片链接的时效性**  
   很多配图 Agent 返回的是 CDN 临时链接，几分钟或几小时后就会失效。所以归档动作必须在拿到 URL 后立刻下载，不能延后到发布之后再补。我踩过的坑是先用 markdown 里的原始链接写进了 `post.md`，发布完再去归档，结果图片已经 404。

2. **并发写入冲突**  
   如果有多个 Agent 同时生成同一篇帖子的不同图片，并行归档可能导致 `assets.json` 被覆盖。解决办法是让归档步骤串行化（每个 post_id 同一时间只有一个写操作），或者使用文件锁。我用的是简单目录锁：先在 `post_dir` 下创建 `.lock` 文件，归档完成再删除，其他协程看到锁则等待。

3. **相对路径一致性**  
   必须要确保 `post.md` 中的图片引用和实际存放路径一致，否则本地预览时图片无法显示。我的做法是在生成 markdown 时就替换所有图片链接为相对路径 `./images/xxx`，而不是保留远程 URL。

4. **单一大 JSON 文件的膨胀问题**  
   最开始我想把所有帖子的 `assets.json` 合并到一个全局文件中，结果随着帖子数量增长，文件变得巨大，每次更新都要全量读写，效率低下。最终还是改为每帖一个 `assets.json`，再用一个极简的索引脚本生成总目录 HTML，性能立刻好转。

5. **隐私与密钥安全**  
   归档内容有时会包含内部链接、测试用的 API key 或者未脱敏的数据，如果整个 `posts/` 目录被直接推送到公共 GitHub 仓库，会出安全事故。建议在写入前做一次敏感信息过滤，并在 `.gitignore` 中明确排除敏感目录。

## 可复用建议

- **把归档步骤收拢为一个 MCP Tool**：无论你的流水线怎么变，只要在发布前调用同一个 `archive` 工具，就能保证结构统一。工具内部处理路径、命名、锁、下载等细节。
- **搭配 Git 做版本控制**：每完成一次归档，自动 `git add` 并 commit。这样你能看到一篇帖子从初稿到最终版的所有变更，回滚到任意历史版本都非常方便。
- **用本地预览脚本提升体验**：写一个简单的 Node.js 或 Python 脚本，读取所有 `assets.json`，生成一个可搜索的本地索引页面，列出标题、日期、摘要、标签，点击即可打开完整 markdown（图片正常显示）。这比在文件管理器里翻找高效得多。
- **将 assets.json 作为下游输入**：比如你可以写一个“二次加工” Agent，只扫描 `assets.json` 中未发布到某个平台的帖子，自动补发。这种链式利用让归档从“死数据”变成“活水源”。

## 总结

自动发帖流水线的本地归档，不是一个炫技的功能，而是保证内容资产长期可维护、可追溯的工程底线。用“一个目录 + post.md + images + assets.json”这种朴素结构，配合少量防御性代码，就能解决 90% 的混乱。投入产出比非常高，尤其是在你需要审计 Agent 输出质量、恢复丢失图片或跨平台复用内容时，它的价值会被成倍放大。

如果现在你的流水线还没有这个环节，建议花一个小时把它加进去——你未来的自己一定会感谢这个决定。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/5947573ae3dbf99a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/476686236a4fe5e9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/3f6949b034170e01.png)

