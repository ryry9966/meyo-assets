---
title: 自动发帖流水线的本地归档设计：从文章、图片到 assets.json
feedId: 29508
source: 综合讨论
publishedAt: 2026-07-18
---

## 背景：流水线产出需要可信的“本地副本”

在 OpenClaw 或类似 Agent 驱动的自动发帖流水线中，一篇文章从生成到发布通常经历：主题抓取 → 大纲生成 → 正文撰写 → 配图生成 → 多平台分发。每一步都可能产生中间产物——Markdown 原文、AI 生成的图片、元数据 JSON、发布记录等。如果这些资产只留在插件缓存或临时目录里，两个严重问题会很快暴露：

1. **不可复现**：下次想重新发布、修改或回溯某篇文章，原始资产已经丢失或路径失效。
2. **协作困难**：多人维护同一套自动化流程时，没法用 Git 等版本工具对内容本身做差异对比。

因此，我们需要一套简单、稳定、不依赖特定平台的本地归档方案，把文章、图片和它们之间的引用关系以结构化方式保存在磁盘上，让任何环节都可以还原出完整内容。

## 问题：资产散落，引用断裂

一个典型痛点：AI 生成正文时嵌入的图片路径是临时 URL 或 `/tmp/midjourney-output/xxx.png`。如果你直接把这个 Markdown 存下来，第二天再打开，所有图片都会变成死链。即便你手动把图片复制到某个文件夹，正文中的引用路径仍然是旧地址，除非你再去修改 Markdown 里的 `![img]` 链接。

更隐蔽的问题是**并发写入**。如果你的流水线是并发生成多篇文章，脚本同时往同一个 `images/` 目录写文件，命名冲突可能覆盖图片，而 assets.json 的更新也有竞态风险。

## 做法：三层归档结构

我采用一套稳定的本地归档模式，目录结构如下：

```
archives/
└── 2025-03-21-why-agent-pipeline-needs-checkpointing/
    ├── article.md
    ├── images/
    │   ├── cover.png
    │   ├── diagram-flow.png
    │   └── ...
    └── assets.json
```

- **目录命名**：`{YYYY-MM-DD}-{slug}`，既保证时间可读，又不会因标题重复而冲突。
- **article.md**：最终可发布的 Markdown 文本，**图片引用全部使用相对路径**（`./images/cover.png`）。这样无论目录被复制到哪里，路径都能正确解析。
- **images/**：存放该文章所有二进制图片资产。
- **assets.json**：记录资产清单与元数据，用于程序化消费，例如后续重新上传到图床或 CDN 时做映射。典型结构：

```json
{
  "article_slug": "why-agent-pipeline-needs-checkpointing",
  "created_at": "2025-03-21T10:30:00Z",
  "images": [
    {
      "file": "images/cover.png",
      "alt": "文章封面图",
      "source_prompt": "A robot archiving documents in a library...",
      "generated_by": "dalle-3"
    }
  ],
  "source_workflow": "openclaw-publish-v2"
}
```

### 自动化归档的集成步骤

1. **接住生成器的输出**：Agent 生成完正文和图片后，不要直接发布。先在临时工作区（例如 `/tmp/pipeline-run/`）完成组装。
2. **归一化图片引用**：用脚本扫描正文中的 `![](...)` 语法，将绝对路径或临时路径替换为 `images/` 下的相对路径，同时把图片文件拷贝到统一目录。
3. **生成 assets.json**：从图片文件名、生成提示词、生成模型等上下文中构建 JSON，记录映射关系。
4. **整体搬入归档目录**：用 `shutil.move` 或 `fs.rename` 将临时文件夹重命名为目标归档路径（同文件系统内原子操作）。
5. **提交到 Git**：将归档目录下的文件 `git add` 并提交，commit message 使用 slug 与日期。

下面是一个最小化可运行的 Python 片段，用于替换图片路径并生成 assets.json（示意，非完整工程代码）：

```python
import re, shutil, json
from pathlib import Path

def archive_article(md_path: Path, img_src_dir: Path, archive_root: Path, slug: str):
    dest = archive_root / slug
    dest.mkdir(parents=True, exist_ok=True)
    img_dest = dest / "images"
    img_dest.mkdir(exist_ok=True)

    content = md_path.read_text(encoding="utf-8")
    assets = {"article_slug": slug, "images": []}

    def replace_img(match):
        alt, src = match.group(1), match.group(2)
        src_path = Path(src)
        if not src_path.is_absolute() and not src_path.exists():
            # 尝试在 img_src_dir 下寻找
            src_path = img_src_dir / src_path.name
        if not src_path.exists():
            return match.group(0)  # 无法处理，保留原样
        new_name = src_path.name
        shutil.copy2(src_path, img_dest / new_name)
        assets["images"].append({
            "file": f"images/{new_name}",
            "alt": alt,
            "original_src": src
        })
        return f"![{alt}](images/{new_name})"

    new_content = re.sub(r'!\[([^\]]*)\]\(([^)]+)\)', replace_img, content)
    (dest / "article.md").write_text(new_content, encoding="utf-8")
    (dest / "assets.json").write_text(json.dumps(assets, indent=2, ensure_ascii=False), encoding="utf-8")
```

## 踩坑点

1. **路径处理陷阱**  
   Markdown 中的图片路径可能是 OS 特有的反斜杠（Windows）或绝对路径。正则处理前务必规范化为 `Path` 对象，再做存在性检查。如果图片本身是从网络 URL 下载的，建议下载后一律重命名为语义化文件名（如 `cover.png`），而不是保留 `a3f2b1c…` 这种无意义哈希。

2. **并发归档的竞态**  
   如果多个流水线实例同时写入同一归档目录，可能出现同 slug 冲突。简单方案是使用锁或唯一 UUID 暂存目录，最后原子重命名。也可以让 slug 带有 uuid 后缀，但会牺牲可读性，偏好上我更倾向在调度层保证串行。

3. **Git 仓库膨胀**  
   图片频繁改动会使 Git 历史迅速变大。建议归档目录单独作为一个 Git 仓库，并开启 Git LFS 管理 `images/` 下的二进制文件，或者定期用 `git gc` 清理。如果图片完全不可变，可以直接在 `.gitignore` 排除，仅保留 assets.json 和 article.md 进行文本版本控制。

4. **assets.json 的“权威性”陷阱**  
   不要试图让 assets.json 成为内容生成的真相来源，它只是记录快照。一旦手动修改了 article.md 中的图片引用，assets.json 就会过时。如果有需要，可以增加一个校验脚本，对比 Markdown 引用的图片列表与 JSON 中记录的差异，发出警告。

## 可复用建议

- **保持归档与发布解耦**：归档负责保存全量“源材料”，发布平台只消费 article.md 和上传后的图片 URL。不要为了发布方便而将归档路径设计成与图床 URL 绑定。
- **注入 Agent 工作流**：在 OpenClaw 里可以将上述归档步骤封装为一个 MCP 工具或插件，接受生成结果和 slug，直接执行本地落盘并将 slug 返回。这样下游的发布步骤就可以依赖归档完成的信号。
- **建立检索索引**：当归档目录积累到几十篇后，可以写一个简单的脚本扫描所有 `assets.json`，生成一个汇总的 `index.json`，方便快速查找某篇文章的发布时间、所用图片等，用于统计或审计。

## 总结

自动发帖流水线的价值不只在于“快”，更在于可维护、可追溯。本地归档层把生成的文章与图片固化为文件系统上的可版本化资产，让每次自动发布都有据可查。通过统一的目录约定、相对路径引用和 assets.json 元数据，我们既降低了断链风险，也为后续的二次分发、合规审查提供了干净的输入源。这种做法不依赖特定云服务，非常适合个人或小团队在 OpenClaw 生态中自建长期可用的内容工作流。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-18/2318d96354a60eeb.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-18/811fe1cb9f537c3c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-18/0b386aa2ed4a08aa.png)

