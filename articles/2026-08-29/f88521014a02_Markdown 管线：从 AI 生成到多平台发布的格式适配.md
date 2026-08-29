---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 35188
source: 综合讨论
publishedAt: 2026-08-29
---

## 背景

在 OpenClaw 这类 Agent 工作流中，AI 生成的内容大多以 Markdown 输出。但 Markdown 只是中间格式，并不等于各平台可发布的最终格式。公众号、知乎、掘金、GitHub、Notion 对标题层级、代码块、公式、表格、图片链接、front matter 的处理差异很大。如果每个任务都各自做格式调整，会出现同一个内容反复手动修改、Agent 输出不稳定、发布结果不可预期的问题。

## 问题

AI 原始 Markdown 常见的问题包括：多了一层 ````markdown` 外层围栏；front matter 被平台当成正文；图片使用本地相对路径或不可外链的原始 URL；代码块缺少语言标注；嵌套列表使用 Tab 或 4 空格导致不同平台渲染不一致；表格对齐不规范；脚注、任务列表、raw HTML 等非标准语法在部分平台不支持。这些问题如果直接在发布前人工处理，成本高且容易遗漏。

## 做法 / 步骤

### 1. 约束生成

给 Agent 的 system prompt 或工具描述中明确输出契约：只输出纯 Markdown 正文，不要外层代码围栏；front matter 仅允许 `title`、`summary`、`tags`、`slug` 等字段；图片使用可访问的 https URL 或统一占位符；标题从二级开始；列表缩进统一 2 空格；代码块必须带语言标注。

这样做比事后清洗更可靠，因为 Agent 在生成时就能遵守边界。

### 2. 解析与校验

用 unified/remark 或 markdown-it 将 Markdown 解析成 AST，不建议使用正则处理 Markdown。校验规则可以包括：

- 顶层只有一个 H1 或没有；
- 所有图片有 alt 和 https 源；
- 所有链接为绝对 URL 或内部锚点；
- front matter 能被正确解析；
- 代码块有语言字段；
- 禁止未转义的 raw HTML 和不支持的脚注语法。

校验失败时返回结构化错误信息，让 Agent 重写或自动修复，而不是直接进入发布流程。

### 3. 平台 profile 适配

为每个平台定义独立的转换规则，例如：

- **公众号**：剥离 front matter；代码块转成带样式的区块或纯文本；宽表格转 HTML 或图片；图片必须上传到素材库并替换 URL；不支持脚注，转为括号注释。
- **知乎**：公式需要转图片或使用知乎支持的格式；代码块保留 GFM；表格要检查移动端宽度。
- **掘金 / GitHub**：支持 GFM、任务列表和表格，但需要处理 front matter 和标题锚点。
- **Notion**：转换为 blocks 或 markdown 导入格式；注意 Notion 对多级代码块高亮支持有限，可能需要拆分。

平台差异不应该散落在脚本各处，而应集中到 profile 配置中。

### 4. 发布执行

可以通过 MCP server 暴露 `publish` 工具，Agent 调用时传入 profile 和最终 Markdown。内部先做 dry run，生成预览 diff，确认后调用平台 API 或剪贴板导入。发布结果写回 Markdown 的 front matter，追加 `platform_links` 字段，生成发布报告。

### 5. 失败处理

平台 API 限流、图片上传失败、登录态过期等情况需要重试和告警。原始 Markdown 必须保留，避免发布失败后无法重发。

## 踩坑点

- 直接对 Markdown 做字符串替换很容易破坏链接、代码块和表格。比如把标题前的 `#` 与代码块中的 `#` 混淆。必须基于 AST 节点处理。
- 外层 ````markdown` 围栏很常见。剥离时要判断首尾 token，不能误删内部代码块围栏。
- front matter 不是标准 Markdown 的一部分，很多平台会原样显示。转换前要显式剥离，并把 `title` 等字段映射到平台自己的标题字段。
- 图片外链容易被防盗链或过期。统一图床 + 上传后替换 URL，不要依赖 AI 给出的原始 URL。
- 公式处理成本最高：知乎和公众号对 LaTeX 渲染支持有限。最好在生成阶段就要求 AI 避免复杂数学公式，或使用图片替代。
- 代码块语言标注丢失在公众号尤其明显，需要提前检查并转换成可读文本。
- 平台间表格样式差异大，宽表格在移动端会被截断。可在转换时设置最大列数或转图片。

## 可复用建议

- 固定 Markdown 子集，超出子集的语法在入口就拦截，避免生成后再修。
- 维护平台 profile 配置，新增平台只加配置和少量转换函数，不改主流程。
- 提供 MCP 工具接口，让 Agent 可以在一次 tool call 内完成校验、转换、发布。
- 发布前永远有 dry run 和 diff，可以看到每个平台的最终效果。
- 保留源 Markdown 和每次转换后的产物，方便回滚和审计。

## 总结

Markdown 管线解决的不是简单的“格式转换”，而是降低 AI 生成内容到多平台发布的不可控性。通过约束输出、AST 校验、profile 适配和发布回写，可以把返工从每次手动处理变成一次配置。对 OpenClaw / Agent / MCP 用户来说，这条管线更适合作为可复用工具链，而不是一次性脚本。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/b685b170912aa721.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/1df2526afe88a3fd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/0909f55d122d5f20.png)

