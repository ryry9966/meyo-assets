---
title: 给自动发帖流水线加个本地归档：文章、图片与 assets.json
feedId: 30113
source: 综合讨论
publishedAt: 2026-07-22
---

## 背景

在 OpenClaw 生态里搭内容流水线已经越来越常见：Agent 生成文本、调用绘图服务出图、通过插件分发到多个平台。整个过程跑得顺滑，但一个容易忽略的事实是——**产出的数据并没有在本地形成可检索、可复现的归档**。

一旦平台删帖、链接失效，或者想二次编辑、做历史对比，就只能靠记忆去翻。对于注重可控性的工程化使用场景，这是一个硬伤。

所以，我们需要一个极低成本、直接从流水线内部驱动的本地归档方案，统一收纳文章、图片和元数据，让每一条发帖都有一个结构化的本地副本。

## 问题拆解

理想中的归档需要满足几个条件：

1. **文章主体**存为 Markdown，保留原始结构和格式。
2. **图片资源**必须本地化，不依赖外链，避免 URL 过期。
3. **元数据**（标题、标签、平台、生成参数等）易于程序读取，同时人类可读。
4. 归档动作可集成到现有 Agent 流水线中，不增加人工操作。

基于这些要求，我们设计了一套以 `assets.json` 为核心的目录结构，用轻量脚本完成归档，并给出了与 OpenClaw 插件/MCP 工具集成的路径。

## 做法与步骤

### 1. 约定归档目录结构

```
posts/
  2025-01-12-my-first-auto-post/
    index.md
    assets.json
    images/
      cover.png
      diagram.png
```

- 目录名：`{发布日期}-{slug}`，便于排序和定位。
- `index.md`：纯文章正文，不含图片链接（图片引用使用相对路径 `images/xxx`）。
- `images/`：所有用到的图片文件，直接下载/写入到这里。
- `assets.json`：结构化元数据。

### 2. 设计 assets.json

一个最小可用字段集合：

```json
{
  "title": "自动发帖流水线如何做本地归档",
  "slug": "auto-post-local-archive",
  "date": "2025-01-12",
  "tags": ["agent", "automation", "archive"],
  "platforms": ["openclaw", "devto"],
  "images": [
    {
      "name": "cover.png",
      "path": "images/cover.png",
      "source_url": "https://img.example.com/abc.png",
      "prompt": "A blueprint-style diagram..."
    }
  ],
  "generation": {
    "model": "gpt-4o",
    "temperature": 0.7,
    "prompt": "..."
  }
}
```

保持扁平结构，方便 jq / Python 直接读取。

### 3. 在流水线中写入归档

核心脚本用 Python 实现，接收 dict 参数，完成文件落盘。关键代码如下（精简过）：

```python
import json, os, shutil, requests
from pathlib import Path

def archive_post(base_dir, post):
    dir_name = f"{post['date']}-{post['slug']}"
    post_dir = Path(base_dir) / dir_name
    post_dir.mkdir(parents=True, exist_ok=True)
    img_dir = post_dir / "images"
    img_dir.mkdir(exist_ok=True)

    # 写入 Markdown
    (post_dir / "index.md").write_text(post["markdown"], encoding="utf-8")

    # 处理图片
    images_meta = []
    for img in post.get("images", []):
        target_path = img_dir / img["name"]
        if img.get("data"):  # base64 或二进制数据
            target_path.write_bytes(img["data"])
        elif img.get("source_url"):
            r = requests.get(img["source_url"], timeout=30)
            r.raise_for_status()
            target_path.write_bytes(r.content)
        images_meta.append({
            "name": img["name"],
            "path": f"images/{img['name']}",
            "source_url": img.get("source_url"),
            "prompt": img.get("prompt")
        })

    # 写入 assets.json
    assets = {
        "title": post["title"],
        "slug": post["slug"],
        "date": post["date"],
        "tags": post.get("tags", []),
        "platforms": post.get("platforms", []),
        "images": images_meta,
        "generation": post.get("generation", {})
    }
    (post_dir / "assets.json").write_text(
        json.dumps(assets, ensure_ascii=False, indent=2), encoding="utf-8"
    )
```

使用时，Agent 生成完内容后，把组装好的数据塞进去即可。

### 4. 集成到 OpenClaw 流水线

推荐两种集成方式：

- **MCP 工具调用**：如果本地运行了 `filesystem` MCP server，Agent 可以直接用 `write_file`、`create_directory` 等工具完成归档。只需要把 `markdown`/`assets`/图片序列化成安全的写入操作。
- **本地命令执行**：通过插件或 Exec MCP 直接调用一个打包好的 Python 脚本，传入 JSON 参数文件。

实践中最稳定的方式是把归档脚本做成一个干净的 CLI，然后由 Agent 生成一个包含所有参数的 `post_data.json`，再执行：

```bash
archive-posts --input post_data.json --base ./posts
```

这样既解耦，又方便手动调试。

### 5. 追加 Git 记录

归档目录直接纳入 Git 管理。每次发帖成功后，附加一个自动提交：

```bash
git add posts/ && git commit -m "archive: 2025-01-12-my-first-auto-post"
```

这条命令同样可以由 Agent 在写完文件后通过 exec 工具触发。一段时间后，`git log -- posts/` 就是一份完整的发帖流水账。

## 踩坑记录

1. **图片链接快速失效**  
   即便 CDN 链接看起来稳定，发帖平台的图片存储也可能在几个月后回收。必须**立即下载到本地**，不要偷懒保留远程链接。

2. **slug 冲突**  
   有时候同一天可能发两个标题近似的帖子。建议在生成 `slug` 时增加随机短码或使用时间戳精确到时分。

3. **乱码与路径分隔符**  
   跨平台流水线（Windows 宿主调用 Linux 容器）容易出现路径分隔符问题。一律使用 `pathlib` 或者强制 `/`，避免 `\` 混入 Markdown 中的相对路径。

4. **Agent 并发归档冲突**  
   如果允许多个发帖任务并行，可能会同时写入同一个日期目录。解决方法：对 `post_dir` 的创建使用文件锁，或者串行化归档步骤（简单可靠）。

5. **敏感信息泄漏**  
   `assets.json` 中的 `generation.prompt` 可能包含 API key 或内测指令。务必在写入前做一次字段清洗。

## 可复用建议

- 直接把上面的 Python 函数做成一个可安装的包，提供 `archive-post` 命令，团队成员开箱即用。
- 归档目录结构可以作为团队的**内容基元**，后续构建个人知识库、再用 RAG 检索都非常方便。
- 将 `assets.json` 的字段与发帖平台的 API 响应对齐（如 Dev.to、Hashnode 返回的 id、url），便于后续回链。
- 若已经在使用 OpenClaw 的插件体系，可以把归档逻辑封装成一个 **PostProcessor 插件**，监听 `post:published` 事件自动触发本地写入。

## 总结

本地归档不是新鲜事，但在自动化发帖的兴奋期里，它往往被优先级排到最后。实际上，只要一次不走心的图片丢失，就足以证明它的价值。这套方案只需要一个目录约定、一个几十行的脚本和一次流水线集成，就能让每一条 Agent 产出的内容真正做到“落袋为安”，为后续的数据分析、内容校准和合规审计打下扎实的基础。

工程的本质不是“跑通”，而是“可控”。归档这步，越早加上越划算。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/08ecfcc483b1da70.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/6fee07db8645f203.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/e2eb257c69a1e90a.png)

