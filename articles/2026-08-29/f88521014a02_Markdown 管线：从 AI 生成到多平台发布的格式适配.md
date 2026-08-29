---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 35189
source: 综合讨论
publishedAt: 2026-08-29
---

# Markdown 管线：从 AI 生成到多平台发布的格式适配

## 背景

在 OpenClaw 的 Agent 工作流里，让模型生成 Markdown 内容并不难，难的是把同一份内容发布到公众号、知乎、掘金、Notion、GitHub 等平台时，各家对 Markdown 的支持差异会直接破坏排版。如果让 Agent 在提示词里记住每个平台的规则，不仅 context 膨胀，输出也会不稳定。更可靠的做法是把“生成”和“适配”拆开：固定一份 canonical Markdown，再交给确定性的 adapter 转成平台格式。

## 问题

AI 生成的 Markdown 通常有几个典型问题：

- 代码块反引号数量不一致，代码内容里出现 ``` 时容易闭合错位。
- 列表、引用之间的空行和缩进随机，导致不同渲染器结果不一致。
- 习惯性使用脚注、mermaid 图、任务列表，但很多内容平台并不支持。
- 标题层级跳跃、表格列过宽、外链图片被盗链。
- 把平台差异写进提示词，会让模型输出不稳定，也无法复用。

这些问题的本质不是“模型不够强”，而是格式规则应当由确定性代码处理，而不是由生成模型临场决策。

## 做法/步骤

### 1. 定义 single source of truth

先固定 canonical Markdown 子集：只保留标题、段落、有序/无序列表、引用、代码块、链接、图片、表格、分割线。禁用 HTML 块、脚注、mermaid、任务列表等平台兼容性差的语法。用 markdownlint 配置锁住规则，例如 MD001、MD009、MD031、MD040、MD041，保证每次生成都经过同一套校验。

### 2. 生成侧约束

在 Agent system prompt 里给出 canonical 规范和反例，要求模型只输出该子集。生成后先调用 MCP 工具 `validate_markdown`，校验失败时把 markdownlint 输出作为 feedback 让模型重试。这样模型只负责内容，不负责平台适配。

### 3. AST 适配层

不要用正则处理 Markdown。用 unified/remark 生态解析成 AST，按平台规则 transform，再用 remark-stringify 统一空行和列表缩进。每个平台一个 adapter：

- **公众号**：Markdown 转行内样式 HTML；代码块转带背景的 `<pre>`；表格根据列宽决定保留或转列表。
- **知乎**：补齐代码块语言标注；清理外链图片；严格处理列表空行。
- **掘金**：单独注入 frontmatter；处理摘要分隔与标签。
- **Notion**：标题从 H2 开始，避免 H1；移除脚注等不支持的语法。

### 4. 接入 OpenClaw

在插件里暴露三个工具：`validate_markdown`、`transform_for_platform`、`preview_diff`。Agent 生成内容后先 validate，通过后 transform，发布前用 preview_diff 查看转换前后差异。转换产物落盘保存，便于发布后出问题时回滚。

## 踩坑点

- **正则解析 Markdown 基本不可靠**：代码块里的 `#`、链接中的括号、嵌套列表都会误判。务必走 AST。
- **公众号不是标准 Markdown 解析器**：直接粘贴代码块和表格会乱。尤其代码内容中出现连续反引号时，不能简单合并反引号，需要根据内容中最长连续反引号数量选择更长的围栏，或者将内容转成 HTML 实体。
- **表格在移动端容易溢出**：公众号、Notion 移动端最好转成列表或提前截断列。
- **AI 经常出现标题跳级**：例如 H2 直接到 H4。canonical 校验应禁止标题层级跳变。
- **frontmatter 不是所有平台都支持**：关键信息不要只放在 canonical 的 frontmatter 里，adapter 按平台单独添加。
- **外链图片可能被平台屏蔽**：发布前需要将图片转存到平台素材库或多平台 CDN。

## 可复用建议

- 把 adapter 写成独立 package，每个平台保存 input/output 样例。
- 回归测试用 golden files：为每个平台保存期望输出，改动后直接 diff。
- 发布前跑 pre-commit：markdownlint + 自定义规则，不通过不允许 transform。
- 保持规则确定性，不让模型参与平台适配。模型只负责内容，格式交给代码。
- MCP 工具返回结构化结果：`{platform, ok, errors, output, diff}`，方便 Agent 根据错误分支处理。

## 总结

Markdown 管线不是一堆正则替换堆叠，而是“单一源规范 + AST 适配层 + 回归测试”。在 OpenClaw 里把生成与格式解耦，可以有效降低 AI 输出的随机性，让多平台发布从反复调格式变成可复现的工程流程。平台差异交给 adapter，内容生成交给模型，两边都不越界，管线才能长期维护。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/fd88d35288dccb7a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/37df9fdbe24bc3e2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/548f681cb3fa014a.png)

