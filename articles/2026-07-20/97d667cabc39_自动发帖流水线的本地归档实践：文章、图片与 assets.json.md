---
title: 自动发帖流水线的本地归档实践：文章、图片与 assets.json
feedId: 29707
source: 综合讨论
publishedAt: 2026-07-20
---

## 背景

在基于 OpenClaw 或 Agent 的内容自动化流水线里，我们常常借助 LLM 生成 Markdown 文章，再用绘图模型产出配图，最后通过发布插件推送到不同平台。大多数实践者只关心“发出去没”，产出的文章和图片很快就被临时目录回收或手动清除。等到需要复盘某一天的内容、二次改写，或者满足合规留档要求时，才发现手里只剩社交平台上的截图，原始 Markdown、原始图片和生成元数据早已无从追溯。

## 问题

缺的是把“生成—发布”的线性流程，补上“归档—可溯源”的闭环。没有归档会带来几个具体麻烦：

- 文章和图片散落各处，靠文件名猜测对应关系；
- 没有元数据记录生成时用到的 prompt、目标平台、时间戳；
- 无法一键重新发布或批量转换格式，因为原始资产早已丢失；
- 多流水线并行产出时，甚至连哪条线生成了什么都查不清。

因此需要一套轻量、无外部依赖、可直接嵌入现有脚本的本地归档方案。

## 做法与步骤

### 1. 目录结构约定

以每篇文章为最小归档单元，一个目录包含所有关联文件：

```
archives/
└── 2025-03-15_3f2a1b/
    ├── article.md
    ├── images/
    │   ├── cover.png
    │   └── diagram.png
    └── assets.json
```

目录名采用 `日期_短ID`，ID 可以是 UUID 前 6 位或递增编号，避免标题中的特殊字符导致路径问题。

### 2. assets.json 设计

这是整个归档的“清单文件”，用于人可读、程序可解析。最小可用结构如下：

```json
{
  "post_id": "2025-03-15_3f2a1b",
  "created_at": "2025-03-15T08:22:00Z",
  "title": "自动归档的最佳实践",
  "platforms": ["twitter", "blog"],
  "prompts": {
    "article": "你是一个技术作者...",
    "image": "A clean archiving pipeline diagram"
  },
  "files": {
    "article": "article.md",
    "images": ["images/cover.png", "images/diagram.png"]
  },
  "tags": ["automation", "archiving"],
  "version": 1
}
```

可根据需要扩展生成模型名称、token 用量、发布状态等字段。关键是每次生成完马上写入，而不是发布后再补。

### 3. 在流水线中集成

用一段伪代码示意集成点：

```python
from pathlib import Path
import json, shutil, uuid
from datetime import datetime

def archive_post(title, markdown_content, image_paths, meta):
    post_id = f"{datetime.now().strftime('%Y-%m-%d')}_{uuid.uuid4().hex[:6]}"
    base = Path("archives") / post_id
    base.mkdir(parents=True, exist_ok=True)

    # 写文章
    (base / "article.md").write_text(markdown_content, encoding="utf-8")

    # 拷贝图片
    imgs_dir = base / "images"
    imgs_dir.mkdir(exist_ok=True)
    dest_images = []
    for src in image_paths:
        dest = imgs_dir / Path(src).name
        shutil.copy2(src, dest)
        dest_images.append(str(dest.relative_to(base)))

    # 写资源清单
    asset = {
        "post_id": post_id,
        "created_at": datetime.utcnow().isoformat() + "Z",
        "title": title,
        **meta,
        "files": {
            "article": "article.md",
            "images": dest_images
        }
    }
    with open(base / "assets.json", "w", encoding="utf-8") as f:
        json.dump(asset, f, ensure_ascii=False, indent=2)

    return base
```

这段逻辑可以封装为 OpenClaw 的 Tool 或 MCP 服务，让 Agent 在生成内容后主动调用。

### 4. 触发时机

建议在内容生成完成、图片下载到本地后立刻归档，而不是等到发布成功。这样可以防范发布失败却没有任何留档的风险。归档目录集中放在项目根目录的 `archives/` 下，可以定期打包、备份或同步到对象存储。

## 踩坑点

1. **文件名非法字符**：标题绝不能直接做目录名，容易因斜杠、冒号、换行导致创建目录失败。统一使用 ID 命名。
2. **图片资源生命周期**：若生成图片为临时 URL，需先下载到本地再复制，否则归档里只有无效引用。
3. **并发冲突**：多条流水线并行生成时要使用唯一 ID（UUID），避免目录重名互相覆盖；`assets.json` 写入需要保证同一目录在单次调用内完成。
4. **版本覆盖策略**：同一篇文章可能多次编辑重新生成，可保留旧版本采用递增版本号，或每天更换新 ID 避免覆盖。不推荐直接覆盖原归档，容易丢失历史记录。
5. **大文件**：如果流水线开始产出视频，本地归档会迅速膨胀。此时可考虑只保留元数据和缩略图，原文件上传至云存储后在 `assets.json` 中记录 URL。

## 可复用建议

- 将上述归档函数抽成标准库或 MCP Server，引入统一接口 `save_archive()`，后续所有内容生成流水线复用同一套归档逻辑。
- 与日志系统打通，归档成功/失败都产出结构化日志，便于监控。
- 将生成时使用的 prompt、模型参数、API 响应一起写入 `assets.json`，方便成本核算和 prompt 迭代。
- 若使用 Git，可对 `archives/` 目录做定期提交，形成自然版本历史。

## 总结

本地归档看起来像是“多做了一步”，但它是自动化内容流水线从实验室走向工程化的分水岭。通过 `文章 Markdown + 图片 + assets.json` 的统一结构，我们不仅保留了所有数字资产，也保留下每次生成过程的上下文。后续无论是二次发布、数据统计、合规审查还是作为新的训练语料，都能直接从归档中提取，而不需要再去各个平台“考古”。克制地做好这一步归档，能为后续的可维护性省下大量时间。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-20/ec1d73691dec65c7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-20/3c476a2058f82c52.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-20/8fda2eb72e9f2105.png)

