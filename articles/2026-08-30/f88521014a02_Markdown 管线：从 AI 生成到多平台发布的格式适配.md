---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 35388
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景

Agent 输出 Markdown 往往只是半成品。常见链路是让模型写文章、周报或发布稿，然后直接发到公众号、Notion、知乎或 WordPress。看起来是“标准 Markdown”，发布后却出现代码块样式丢失、表格被压缩、图片 403、脚注残留、标题层级混乱。

根因不是模型写得差，而是 Markdown 方言太多，AI 生成又不够稳定。多平台发布必须把“AI 原文”先收敛成结构化中间态，再做平台适配。

## 问题拆解

1. **方言差异**：GFM 表格、任务列表、脚注、公式、mermaid 在各平台支持不同。
2. **生成不稳定**：全文被 ```markdown``` 包裹、frontmatter 污染、标题越级、空段落。
3. **资源不可用**：本地路径、防盗链、webp/svg 限制。
4. **元数据混入正文**：模型开场白“以下是正文”等进入发布内容。

## 做法：三段式管线

### 1. 规范化：收敛为 canonical markdown

建议以 CommonMark + GFM 表格/任务列表/围栏代码块/链接图片为中间态，前置 frontmatter 存标题、标签、摘要、封面。

```yaml
---
title: 周报
tags: [release, agent]
summary: 本周发布摘要
cover: https://cdn.example.com/cover.png
---
```

规范化步骤：

- 用 AST 解析，不要正则替换。
- 整篇外层如果是 ```markdown ...```，先剥离代码围栏。
- 标题从 H2 开始重排，避免和页面标题重复。
- 删除 frontmatter、空段落、模型开场白/结束语。
- 本地图片上传对象存储，替换为 CDN 绝对 URL；链接统一规范。

### 2. 适配器：canonical AST -> 目标格式

每个平台独立 adapter，不要共用一套 HTML。

- **公众号**：用 markdown-it 渲染 HTML 并内联样式；代码块转 `pre` 浅灰背景；表格单元格超限合并；图片转 JPEG/PNG，不用 webp。
- **Notion**：节点映射到 paragraph/heading/code/table/quote 等 block，图片先上传再挂 block。
- **知乎/掘金**：保留 GFM，公式和脚注降级为图片或括号文本。
- **WordPress**：HTML 或 Gutenberg blocks；外链图片下载入媒体库。

示例配置：

```yaml
pipeline:
  normalize:
    strip_outer_fence: true
    heading_start: 2
  adapters:
    wechat:
      image_format: jpeg
      inline_style: true
    notion:
      max_table_cell_chars: 2000
```

### 3. 校验与回滚

发布前 dry-run 并校验：空标题、断裂链接、本地路径、图片可访问性、表格超限、正文长度。失败进入人工或重试，不直接覆盖线上。

## 踩坑点

- 不要用正则处理嵌套 Markdown，如引用里的列表、代码块里的表格。
- 公众号代码块必须内联样式，外部 CSS 会被过滤。
- Notion 表格单元格有 2000 字符限制，长文单元格需拆分。
- 外链图片直接发公众号常见 403，需下载上传到自己的 CDN。
- 模型输出的 HTML 注释、零宽空格、`<br>` 要清理。
- 多平台发布时，源文档只保留 canonical markdown，不要把平台 HTML 当源文件改。

## 可复用建议

- 拆成 MCP 工具：`md_normalize`、`md_adapt_wechat`、`md_adapt_notion`、`md_validate`，由 agent 按平台调用。
- 统一输出 `{meta, markdown, assets}`，不要只返回裸字符串。
- 准备典型 fixture 库，包含表格、代码块、嵌套引用、多图。
- 加 diff 检查，多平台正文差异应只来自适配，不应改变语义。
- 记录规范化前后 AST 差异，便于排查模型输出变化。

## 总结

AI 生成 Markdown 到多平台发布，核心不是“写更好提示词”，而是建立从不可信文本到结构化中间态的管线：先规范化，再 AST 适配，最后校验回滚。不要让平台特性倒灌回源文档。OpenClaw 的自动化场景里，这套管线可以沉淀为插件或 MCP 工具，降低发布事故和人工返工。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/a11055024b2092c6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/714d415dca8eea7b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/1bc1c92e15ad361d.png)

