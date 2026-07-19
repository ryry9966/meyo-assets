---
title: 自动化发布流水线兜底：本地归档文章、图片与 assets.json
feedId: 29642
source: 综合讨论
publishedAt: 2026-07-19
---

## 背景：发布之后，内容去哪了？

在基于 OpenClaw / Agent / MCP 搭建的自动发帖流水线里，大家通常只关心两件事：怎么让模型写出更好的文章，以及怎么把东西推送到各个平台。发布完成那一刻，这条流水线就被视为“已结束”。

但只要你真正跑过一段时间的自动化，就会意识到缺了关键一环——**本地没有留下可追溯、可复用的内容归档**。文章散落在临时目录，图片引用还是远程 URL，哪怕只是三个月后想查一下某天的发布记录，都可能要翻半天日志、重新拼凑。

这不止是“爱整理”的问题，而是**长期运营必须的基础工程**。如果你希望团队复用历史内容，或者需要审计、回滚、跨平台二次分发，就必须有一套自动化的本地归档能兜底。

## 问题：散乱的产物，无法持续维护

典型的情况是这样的：Agent 生成了一篇 Markdown，配了几张图，调用平台 API 发布成功。但文章内容在内存或临时文件里，图片可能用了某图床的临时外链，用户看不到的元数据（标题、发布时间、slug、平台信息）都没留下来。等你想把这些东西安全地落盘，才发现：

- Markdown 里图片引用混杂：有的是本地临时路径，有的是图床外链，有的甚至干脆是 base64 内嵌。
- 没有统一的资产清单，想知道某篇文章用了哪些图、分别对应什么原始资源，得靠人肉检索。
- 多个 Agent 并发跑时，临时文件极易互相覆盖或丢失。

这些问题本质上源于流水线中缺少一个**“归档步骤”**，让所有生成物以可追溯、可机器读取的形式落地。

## 做法：一套标准化本地归档流程

我们设计了一条轻量级归档流水线，核心产出是一个规范目录，包含三样东西：**整理好的文章 Markdown、所有本地图片副本、以及一个资产元数据文件 `assets.json`**。

### 1. 约定目录结构

以日期 + slug 作为唯一标识，例如：

```
archive/
  └── 2025-03-27-understanding-agent-archiving/
        ├── article.md
        ├── images/
        │   ├── architecture.png
        │   └── flowchart.png
        └── assets.json
```

这种结构既方便人眼浏览，也方便脚本批量遍历。

### 2. 在流水线末端接入归档逻辑

无论你的内容是直接由 Agent 产生，还是通过 MCP Tool 拼接完成，归档步骤只需要在“发布成功”后执行。所需输入包括：

- 文章的完整 Markdown 文本
- 标题、slug、创建时间、目标平台等元信息
- 所有相关图片的原始路径或 URL（Agent 生成的图片往往在某个临时目录或对象存储里）

归档 Task 要做三件事：

- **标准化图片引用**：扫描 Markdown 中的 `![](...)` 语法，将远程 URL 图片下载到 `images/` 下，并将引用路径改为统一的 `images/<filename>`；对已存在的本地文件直接复制。如果下载失败，保留原始引用，同时在 `assets.json` 里标记为 `external`。
- **写入 article.md**：将修改后的 Markdown 写入归档目录。
- **生成 assets.json**：记录所有资源清单与关键元数据。

示例 `assets.json`：

```json
{
  "slug": "understanding-agent-archiving",
  "title": "Understanding Agent Archiving",
  "created_at": "2025-03-27T10:00:00Z",
  "platforms": ["devto", "hashnode"],
  "images": [
    {
      "filename": "architecture.png",
      "alt": "archiving architecture",
      "original_url": "https://cdn.example.com/raw/arch.png",
      "local_path": "images/architecture.png",
      "status": "downloaded"
    }
  ],
  "article_file": "article.md",
  "tags": ["agent", "pipeline", "archiving"]
}
```

这个 JSON 既能被程序解析，又能一眼看出文章资产的全貌。

### 3. 实现建议

在实际工程中，通常会封装为一个 **归档 MCP Server 工具** 或独立 Python 函数，通过标准的 Tool Call 输入参数执行。流程伪代码：

```
def archive_article(markdown: str, meta: dict, image_map: dict, base_dir: Path):
    archive_dir = base_dir / f"{meta['date']}-{meta['slug']}"
    archive_dir.mkdir(parents=True, exist_ok=True)
    img_dir = archive_dir / "images"
    img_dir.mkdir(exist_ok=True)
    
    assets = {"slug": meta["slug"], ...}
    
    for original_path, filename in image_map.items():
        local_path = img_dir / filename
        copy_or_download(original_path, local_path)
        markdown = replace_image_ref(markdown, original_path, f"images/{filename}")
        assets["images"].append({...})
    
    (archive_dir / "article.md").write_text(markdown)
    (archive_dir / "assets.json").write_text(json.dumps(assets))
```

如果还在用 Git 管理内容，再补一个 `git add && git commit`，就是完整的版本化归档。

## 踩坑点：不是简单的文件复制

实践过程中有几个容易翻车的地方，值得提前应对：

- **路径分隔符陷阱**：Windows 上是反斜杠，而 Markdown 引用只认正斜杠。硬编码路径拼接会出问题，必须用 `pathlib` 或强制 `/` 分隔。
- **Markdown 引用替换不彻底**：图片语法可能带有 title（`![alt](url "title")`），也可能有空格。正则需要写得足够宽松，且替换后保留 title 等可选部分。
- **图片下载的超时与大文件**：万一图床不可达，不可让归档流程卡死。通常设 10 秒超时 + 最大 10 MB 限制，失败后标记 `status: "external"` 存为原始 URL，而不丢失。
- **并发安全**：高并发 Agent 跑归档时，如果没有唯一目录名（加上时间戳或 UUID 后缀），可能导致文件覆盖。最好在 slug 后追加短哈希。
- **assets.json 的编码与换行**：确保写入 UTF-8，使用 `ensure_ascii=False` 并加换行缩进，方便人读和 diff。

## 可复用建议

归档逻辑本身与业务无关，完全可以抽象为通用组件：

- **做成 MCP Tool**：定义 `archive_content` 工具，接受结构化输入，方便任何 Agent 通过 Tool Call 调用。
- **与 OpenClaw 集成**：在 OpenClaw 的任务流程图中加入“归档”节点，并在编排脚本中自动向该工具传入上下文。
- **配置归档根目录**：允许按环境变量 `ARCHIVE_ROOT` 或传入参数灵活指定，方便本地和 CI 环境复用。
- **支持多种图片源**：既支持本地文件路径，也支持 HTTP(S) URL、base64 内嵌图片的解码，覆盖所有可能情况。

开源社区已有类似的小工具，但自己实现一个 50 行的函数往往更容易适配自己的流水线，也避免依赖地狱。

## 总结

自动化发帖流水线中的本地归档，听起来像“不重要的小事”，实际上是把生成的内容资产真正沉淀下来、可持续运营的基础。**一篇文章、几张图、一份 assets.json**，就可以让你的内容仓库从“草稿流”变成“长期可维护资产”。下次你的 Agent 发布完内容后，别急着结束任务——让它顺便把归档也做了，未来你会感谢现在的自己。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/956d6a44f2a28a7f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/23b6d8b86af40863.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-19/c7cfbcaf264783e0.png)

