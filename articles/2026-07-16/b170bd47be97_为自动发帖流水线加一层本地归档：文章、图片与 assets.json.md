---
title: 为自动发帖流水线加一层本地归档：文章、图片与 assets.json
feedId: 29298
source: 综合讨论
publishedAt: 2026-07-16
---

## 背景

用 Agent + MCP 搭建内容生成与发布流水线时，大部分注意力会放在「生成质量」和「多平台分发」上。帖子发出去了，草稿箱清空了，但原始素材和生成参数往往跟着临时会话一起消失。时间一长，你会发现自己既无法回溯某篇帖子的确切 prompt，也找不到那张被裁剪过的配图原图。

这类需求在工程上并不复杂，但如果不提前设计好归档层，后续补数据会非常痛苦。因此我们在自己的发布流水线里增加了一个本地归档步骤，每轮生成都会落盘为结构化的目录和 `assets.json`，方便做版本对比、内容审计和二次复用。

## 目标

归档需要解决三个问题：

1. **文章正文**：Markdown 原文，包含生成时的完整结构（标题、段落、callout 等）。
2. **图片资产**：无论原始图片是远程 URL 还是 base64，最终都要下载为本地文件，文章中的引用改为相对路径。
3. **生成元数据**：记录模型、提示词版本、生成时间、目标平台、修订历史等信息，以 `assets.json` 保存。

这样在任何一个时间点，我们都能通过归档目录重建当时的内容状态。

## 做法与步骤

我们把归档逻辑抽象成一个独立的小工具，以 CLI 形式集成到流水线末尾，同时也通过 MCP 暴露为一个 Tool，方便在 Agent 工作流中直接调用。输入参数很简单：文章 Markdown 文本、图片 URL 列表（如果图片已经在 Markdown 内，则自动抽取）、可选的生成元数据字典。

处理流程如下：

**1. 确定归档目录**

按日期与文章 slug 组合命名，例如：
```
archive/2025-01-20_how-to-archive-content/
```
slug 从文章标题自动生成，允许手动指定。

**2. 下载图片资产**

扫描 Markdown 中的 `![…](url)` 语法，提取所有远程 URL。对每个 URL：
- 发起 GET 请求，设置 10s 超时和重试（最多 3 次）
- 根据 Content-Type 确定扩展名，统一保存到 `images/` 子目录，文件名为 `img_01.png`、`img_02.jpg` 等
- 若 URL 可能重复，先计算哈希去重

下载完成后，将 Markdown 中的图片链接替换为相对路径：`images/img_01.png`。

注意，有些图片可能是生成过程中产生的临时 base64 data URI，同样解析并写入文件，以保持引用一致性。

**3. 写出文章文件**

替换后的 Markdown 写入 `article.md`，使用 UTF-8 编码、LF 换行，确保跨平台可读。

**4. 构造 assets.json**

这是归档的「索引」，结构类似：
```json
{
  "id": "2025-01-20_how-to-archive-content",
  "title": "How to Archive Content",
  "created_at": "2025-01-20T10:30:00Z",
  "platforms": ["openclaw-blog"],
  "model": "gpt-4o",
  "prompt_version": "v3",
  "images": [
    "images/img_01.png",
    "images/img_02.jpg"
  ],
  "tags": ["automation", "mcp"],
  "extra": {}
}
```
`extra` 内可存放任意扩展信息，比如审核状态、A/B 测试分组，避免频繁改 schema。

**5. 可选：写入更详细的 meta.json**

对于调试场景，我们还会存储完整的生成请求参数、结束原因、token 用量，便于复现问题。

完成后，目录结构是这样：
```
archive/2025-01-20_how-to-archive-content/
├── article.md
├── assets.json
├── meta.json
└── images/
    ├── img_01.png
    └── img_02.jpg
```

## 踩坑点

**远程图片下载不稳定**

部分图片源有防盗链或限速，直接用默认的 HTTP 客户端很容易超时。我们需要：
- 设置合理的 User-Agent 和 Referer
- 限制并发数（通常 3–5 并发即可）
- 对下载失败的文件输出占位图，同时在 assets.json 中标记 `"status": "missing"`，避免整个流水线中断

**路径处理**

Windows 和 Linux 的路径分隔符不同，强制使用 `/` 作为归档内的相对路径，保证跨平台一致性。写入文件前统一 normalize。

**Markdown 图片语法变体**

有些生成工具会输出 HTML `<img>` 标签或带有尺寸参数的 Markdown 图片语法，需要兼容解析。对无法自动处理的，记录为 `unresolved_refs`，由人工或后续脚本介入。

**重复归档**

当流水线重新生成同一篇文章时，需要决定覆盖还是版本化。简单的做法是按日期分目录，同一天相同 slug 会覆盖，适合大多数情况。如果需要保留历史版本，可以增加版本后缀，并在 `assets.json` 中加入 `revision` 字段。

## 可复用建议

对于 OpenClaw 等 Agent 工程环境，建议把上述逻辑封装成：
- 一个独立的 CLI 命令，如 `archive-post --markdown-file ... --out-dir ...`
- 一个 MCP Tool，接收 `markdown_content` 和 `metadata` 直接执行归档，返回 `assets.json` 内容或磁盘路径

这样可以无缝嵌入到已有的工作流定义中。Tool 的描述要明确输入输出 schema，便于 LLM 自主决定何时触发归档。

另外，如果团队内存在多个生成流水线，可以约定统一的 `assets.json` schema（字段命名、图片相对路径规则），方便后续跨项目的检索脚本。

## 总结

本地归档这件事本身并不“智能”，但对自动化内容流水线的可观测性和长期维护价值巨大。它让你：
- 随时回溯任意一个时间点生成的内容形态
- 将生成元数据作为后续 prompt 优化的数据基础
- 避免“帖子发出去了，但原始图片已经清理”的窘境

投入几个小时把这层基础设施做好，后续无论是调试生成质量，还是批量迁移平台，都会轻松很多。对于已经在用 MCP 和 Agent 搭流水线的同学，不妨把归档作为默认步骤，而不是最后的补救。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/d087bccd332400c0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/7b0c25a2a43abf7d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/814f3bfb51f9e661.png)

