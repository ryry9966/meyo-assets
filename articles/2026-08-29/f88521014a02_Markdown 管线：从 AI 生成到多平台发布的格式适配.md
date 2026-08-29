---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 35238
source: 综合讨论
publishedAt: 2026-08-29
---

# Markdown 管线：从 AI 生成到多平台发布的格式适配

## 背景

用 Agent 批量产出内容后，发布环节经常变成人工返工。AI 生成的 Markdown 通常包含 frontmatter、HTML 注释、无语言标注的代码块、相对路径图片、行内 HTML，各平台对这些方言支持不同。直接粘贴到公众号、知乎、Notion、飞书或静态站点，结果往往是表格塌掉、代码换行丢失、标题层级被打乱，甚至 frontmatter 直接出现在正文。

问题不在于 Markdown 本身，而在于“AI 默认输出”与“平台期望输入”之间存在差异。解决办法是在中间加一层规范化与适配管线。

## 问题拆解

1. **AI 输出噪声**：模型喜欢生成 Markdown 代码围栏包裹、HTML 注释、多余空行、emoji、非标准表格、未闭合标签。
2. **Markdown 方言差异**：微信公众号更接近 HTML，表格需要 `table` 而非 pipe table；知乎对代码块和公式支持有限；Notion 对嵌套列表和脚注不友好。
3. **发布侧限制**：部分平台没有公开的 Markdown 渲染接口，只有块 API 或草稿 API，还会过滤行内样式和脚本。

## 管线做法

在 OpenClaw 中拆成四段：解析、清洗、转换、适配发布。

### 1. 解析为 AST

用 `unified` + `remark-parse` 把文本解析成 mdast，不要用正则处理。解析后按节点类型做替换和校验，而不是按字符串匹配。

### 2. 清洗 AI 常见噪声

写一个 `remark` 插件，遍历 AST 处理：

- 删除 HTML 注释和脚本节点；
- 标题限制在 64 字符以内，避免平台截断；
- 代码块补全 `lang`，未知语言降级为 `text`；
- 图片 `src` 改为绝对 URL，并补 `alt`；
- 剥离 frontmatter，不把 YAML 丢回正文；
- 表格统一成 GFM 表格，后续按平台决定保留还是转 HTML。

### 3. 平台适配器

每个平台写一个 adapter，输入统一 mdast，输出目标格式。

- **微信公众号**：用 `remark-rehype` 生成 HTML 并注入内联样式；代码块需要 `white-space: pre-wrap` 和 `word-break: break-all`，否则移动端横向溢出；pipe table 必须转 HTML `table`。
- **知乎**：保留 Markdown，但 `mermaid`、`math` 代码块换成图片或删除；代码语言标注尽量只在常见语言白名单内。
- **Notion**：走 Markdown import，但把脚注转成行内文本，深层嵌套列表拍平。
- **静态站点/仓库**：保留完整 Markdown 和 frontmatter，但重写相对链接，并补 `slug` 和摘要。

### 4. 通过 MCP 发布

发布动作挂 MCP server，例如 `create_wechat_draft`、`append_notion_block`。管线输出标准化的 `{ platform, content, meta }`，MCP 只负责调用 API。这样平台 API 变化时，只改 MCP 适配器，不改清洗与转换逻辑。

## 踩坑点

- 不要用字符串替换做清洗。代码块内的 `#`、表格里的 `|` 很容易被正则误伤，AST 节点处理更稳。
- 未知语言标注可能让平台丢弃代码块。建议维护语言白名单，未知语言降级到 `text`。
- 宽表格在公众号、飞书和移动端会溢出。超过 4 列或单元格过长时，自动转成列表或图片。
- 相对路径图片发布后 404。发布前校验所有 URL，失败则阻断发布或替换占位图。
- frontmatter 在解析阶段就剥离，避免 YAML 泄漏到正文。
- 不要直接发送 AI 首轮输出。用 system prompt 把生成限制在受限 Markdown 子集：禁用 HTML 注释、mermaid、行内 HTML，要求绝对图片 URL 和代码块语言标注。

## 可复用建议

把管线配置写成 OpenClaw 插件，核心是“一个中间表示 + 多个 adapter”。建议保持如下结构：

```text
markdown-pipeline/
  rules/clean-ai-noise.js
  adapters/wechat.js
  adapters/notion.js
  adapters/zhihu.js
  index.js
```

每次发布前执行预检：解析是否成功、图片是否可达、标题长度、代码语言是否在白名单、表格列数是否超标。通过后 dry-run 输出目标格式预览，再调用 MCP 发布。不要依赖 prompt 提醒“适配平台”，而应让内容进入固定工程约束。

## 总结

Markdown 管线的价值不是一次性转换脚本，而是把解析、清洗、平台适配拆开，并让发布动作通过 MCP 做薄封装。这样 AI 侧可以继续迭代生成质量，发布侧不再反复返工。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/9bff088ca0f736bb.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/669604c14066b627.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/6c6ad57f45b781a8.png)

