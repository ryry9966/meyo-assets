---
title: OpenClaw 自动发帖流水线本地归档实践：文章、图片与 assets.json
feedId: 29496
source: 综合讨论
publishedAt: 2026-07-18
---

## 背景

如果你已经在用 OpenClaw 搭建自动发帖流水线——可能是通过 Agent 调用多个 MCP 工具生成内容、配图，最后推送到不同平台——那么很快就会遇到一个工程化问题：**这些生成出来的文章和图片，到底该怎么系统性地存下来？**

刚开始跑流水线时，我们往往只关心“发出去没有”。Markdown 内容留在终端窗口，图片被临时路径引用，发送完即丢弃。直到某一天你需要回溯一周前的帖子，想要复现某个配图风格，或者需要向他人展示“这个系列到底产出了什么”——才发现手里什么都没有。

**自动生成的内容也是资产**。对一条内容流水线而言，归档不应是事后补救，而应是流水线的一个内建步骤。

本文记录我在 OpenClaw 环境下为自动发帖流水线加入本地归档机制的做法，包括 Markdown 正文、图片文件以及描述本次运行上下文信息的 `assets.json`。

## 问题拆解

一条典型的自动发帖流水线大致包含：

1. 内容生成（Agent 调用 LLM 产出标题与正文）
2. 图片生成（调用 Stable Diffusion / DALL·E 等，或通过前端模板服务）
3. 元数据组装（平台参数、标签、发布时间等）
4. 发布动作

在这条链路上，归档需要捕获的信息至少包括：

- 最终的 Markdown 原文（而非原始生成输出，可能经过格式化或字数是调整）
- 所有引用图片的本地副本（避免 URL 失效）
- 本次运行的上下文：用了什么模型、什么参数、生成了多少次、耗时等

这些信息一旦封装成结构化目录，就能支持：

- 快速查找历史文章
- 复现某次生成的具体配置
- 批量导出给其他系统使用
- 作为版本基线，与 Git 仓库配合进行内容变更追踪

## 归档结构与工具选型

### 目录约定

我采用的归档根目录结构如下：

```
posts_archive/
  2025-01-15_an-intro-to-mcp/
    post.md
    images/
      cover.png
      diagram.png
    assets.json
  2025-01-16_optimizing-prompt-cache/
    ...
```

每个子目录以 `日期_英文-slug` 命名，确保可排序且便于人类阅读。`post.md` 存放最终正文，`images/` 存放所有本地化后的图片，`assets.json` 记录生成元数据。

### 使用 MCP 文件系统工具

在 OpenClaw 的流水线编排中，可以直接利用 MCP 的 `filesystem` 插件（如 `@anthropic/mcp-server-filesystem`）来完成目录创建和文件写入。大致步骤：

1. 在生成阶段拿到 `run_id` 或 `slug`，组装目标路径
2. 使用 `create_directory` 工具建立归档目录
3. 将 Markdown 正文通过 `write_file` 写入 `post.md`
4. 将生成的图片二进制或 Base64 通过工具写入 `images/` 目录
5. 组装 `assets.json` 内容并写入

这样归档步骤就变成流水线中的一个 MCP 工具调用环节，与发布步骤解耦。即便发布失败，归档仍然会被执行，避免信息丢失。

## 具体实现步骤

### 1. 定义 assets.json 的规范

`assets.json` 不宜过于复杂，但要覆盖“可复现”所需的最小信息集。我使用的结构如下：

```json
{
  "run_id": "20250115-mcp-intro-3f2a1b",
  "timestamp": "2025-01-15T10:23:00Z",
  "model": {
    "text": "claude-sonnet-4-20250514",
    "image": "stable-diffusion-xl"
  },
  "generation_params": {
    "temperature": 0.7,
    "max_tokens": 2000,
    "image_prompt": "A minimalist diagram of MCP..."
  },
  "platforms": ["openclaw-cn", "discord"],
  "slug": "an-intro-to-mcp",
  "images": [
    { "filename": "cover.png", "prompt": "..." },
    { "filename": "diagram.png", "prompt": "..." }
  ]
}
```

可以根据需要扩展，比如加入关键词、字数统计、生成耗时等。关键在于**记录模型与参数**，这对日后回溯某个排版决策特别有帮助。

### 2. 处理图片引用路径

生成出来的 Markdown 里，图片引用可能是临时 URL 或本地相对路径。归档时需要统一改写为 `./images/xxx.png`，使得归档后的 `post.md` 独立于生成环境可直接渲染。

可以在写入 `post.md` 之前，用简单的字符串替换或正则将外部 URL 替换为本地路径，同时确保图片已经通过下载或复制落入 `images/` 目录。如果你是用 MCP 的 `fetch` 工具获取图片，完全可以直接落盘到归档路径，随后再改写 Markdown。

这里有一个容易忽略的细节：**某些平台在发布时需要原始远程 URL**。因此应该在归档之外保存一份原始引用信息。我一般会在 `assets.json` 里保留 `original_image_urls` 字段，归档 `post.md` 则统一使用本地路径。这样两个需求都能满足。

### 3. 脚本化归档步骤

在 OpenClaw 编排中，可以通过 ReAct 模式将归档作为一个工具调用节点。但为了避免 Agent 复杂度过高，我更倾向于将归档逻辑封装为一个小型独立脚本（Python 或 Node），然后以 MCP 工具或命令执行器的方式暴露出来。

以 Python 脚本为例，核心逻辑只有几十行：

```python
def archive_post(run_id, slug, markdown_text, image_list,
                 metadata, base_dir="posts_archive"):
    # 创建目录
    dir_name = f"{metadata['timestamp'][:10]}_{slug}"
    target = Path(base_dir) / dir_name
    target.mkdir(parents=True, exist_ok=True)
    
    # 写入 post.md（图片路径已改写）
    (target / "post.md").write_text(markdown_text)
    
    # 写入图片
    img_dir = target / "images"
    img_dir.mkdir(exist_ok=True)
    for img in image_list:
        (img_dir / img["filename"]).write_bytes(img["data"])
    
    # 写入 assets.json
    assets = {**metadata, "images": [
        {"filename": i["filename"], "prompt": i["prompt"]}
        for i in image_list
    ]}
    (target / "assets.json").write_text(json.dumps(assets, indent=2))
```

封装为 MCP 工具后，流水线只需要生成好数据，调用该工具即可。

## 踩坑点

1. **路径规范化**  
   Windows 与 Linux 的路径分隔符不一致，建议统一使用正斜杠，并在脚本中做兼容处理。MCP 文件系统工具一般已处理此问题，但自建工具时要留意。

2. **图片写入时的 Base64 编码**  
   如果模型返回的是 Base64 编码的图片，直接写入文件时记得先解码。忘记解码会导致文件损坏，渲染时出现“无法识别的图像”而耗费大量排查时间。

3. **流水线中断导致归档不完整**  
   如果归档步骤本身抛出异常，会导致目录存在但缺少部分文件。建议先写入临时目录，全部完成后重命名至最终位置，或至少添加状态标记。这能避免无效目录堆积。

4. **归档与版本控制**  
   如果希望同时将归档纳入 Git 版本库，要特别注意图片体积。通常只提交 `post.md` 和 `assets.json`，将 `images/` 加入 `.gitignore`，或使用 Git LFS。大量二进制文件进入仓库会让操作立刻变慢。

5. **并发运行时的目录冲突**  
   流水线如果支持并发任务，不同 run 可能同时创建相同 slug。用 `run_id` 或时间戳到秒作为唯一标识可以有效规避。

## 可复用建议

- **将归档工具作为 MCP 独立服务**，而不是紧耦合在某个 Agent 的提示词里。这样无论是通过 Claude、OpenClaw 还是其他编排引擎，都可以复用同一套归档逻辑。
- **生成 slug 的策略**：可以先用 LLM 根据标题生成英文 slug，然后人工或规则检视。自动化度足够高时，直接用日期加 `run_id` 短码也未尝不可。
- **与报告系统联动**：`assets.json` 积累了一定数量后，可以写一个简单的聚合脚本，统计各模型的使用频次、平均生成字数等，为优化流水线提供数据支持。

## 总结

自动发帖流水线的本地归档，看似是一件“预防性”的工作，实际上决定了你对自己产出内容的掌控力。将文章、图片和元数据规范地存放于本地目录中，不仅能避免数据丢失，更让你有能力回溯、复盘和二次加工这些内容资产。

在 OpenClaw 的工具链语境下，依靠 MCP 文件系统或一个轻量自建工具，就能在不增加太多复杂度的前提下把归档固化到每条流水线里。这远比事后重新整理散落的文件要划算得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-18/74598b81b272bb58.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-18/e41c81dc13731608.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-18/ea698bab449f05a2.png)

