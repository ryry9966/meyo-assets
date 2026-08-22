---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 34288
source: 综合讨论
publishedAt: 2026-08-23
---

## 背景

在 OpenClaw 的自动化发布场景里，Agent 生成 Markdown 往往只是第一步。真正的麻烦出现在公众号、知乎、掘金、Notion、GitHub、小红书等目标平台开始“按自己的方式”解释这些 Markdown 时。同一个文件，在 GitHub 上正常，在公众号后台可能变成一坨无样式文本；表格在桌面端好好的，到了移动端直接溢出。把“生成”直接接到“发布”，很容易让自动管线停在最后一公里。

## 问题定位

差异不是 Markdown 本身的问题，而是各平台实现子集不同。常见不兼容包括：换行是否需要两个空格或反斜杠、表格是否渲染、代码块是否保留语言标记和语法高亮、脚注与任务列表是否支持、数学公式是否由 KaTeX/MathJax 处理、图片是否只接受绝对 URL、以及标题与正文间距。

AI 生成内容还会放大这些问题：标题层级跳跃、空列表项、emoji 过多、代码块没有语言、链接是占位符、表格列宽不合理。对 OpenClaw Agent/MCP 来说，如果转换只能靠人工复制到公众号编辑器再手动修，自动化链条就断了。需要一层显式的 Markdown 管线。

## 做法

### 1. 定义规范子集，而不是直接吃原始输出

给生成阶段加约束：只允许 H2/H3、段落、有序/无序列表、代码块、表格、图片、链接、引用、加粗和行内代码。禁止脚注、任务列表、HTML、一级标题多次出现。元数据统一放 frontmatter：

```yaml
title: 示例标题
summary: 摘要
tags: [agent, mcp]
cover: https://cdn.example.com/cover.png
target_platforms: [wechat, zhihu, juejin]
```

这样后续转换不需要处理无穷组合。

### 2. 用 AST 做校验和转换，不要用正则

我会用 unified/remark 把 Markdown 解析成 AST，再跑自定义 lint：检查标题是否跳级、图片是否有 alt 和绝对 URL、代码块是否声明语言、表格列数是否超过 4、链接协议是否为 https、单段是否超过 800 字符。正则在这里不可靠，嵌套列表或代码块里的 `#` 很容易误判。

### 3. 平台适配器独立出来

每个平台一个 adapter，输入统一的规范 Markdown 或 AST，输出该平台需要的格式。

- **公众号 adapter**：markdown-it 渲染 HTML → 清洗 style → 输出可粘贴到公众号后台的富文本；代码块若高亮丢失，则改为生成图片插入。
- **知乎/掘金 adapter**：保留 GFM 子集，将表格转为 HTML 表格，图片链接改写为绝对地址。
- **Notion adapter**：按块转换，或走 API 创建 blocks，避免导入时吞换行。
- **小红书 adapter**：不假装支持 Markdown，提取文本段落，配合封面/卡片图生成纯文本和图片资产。

### 4. 预览和回归

本地起一个 preview 命令，按目标平台渲染 HTML 并截图/对比。维护 golden cases：一个覆盖标题、列表、表格、代码块、图片、链接、引用的标准 fixture，每次改 adapter 后跑一遍，避免某个平台格式意外回退。

## 踩坑点

- 公众号后台二次编辑会改写 HTML。不要把它当作存储，发布态由管线重新生成。
- 代码高亮不要依赖复制粘贴。公众号粘贴后大概率退回纯文本，代码块转图片或使用能保留内联样式的格式化方式更稳。
- 移动端表格很难看。超过 4 列的表格在手机端必然溢出，最佳做法是限制列数，或在适配器中把宽表自动转成“字段：值”列表。
- AI 容易生成空列表项和无语义空格，lint 要显式禁止空项，并在生成 prompt 中加入反面示例。
- 相对路径图片/链接在平台基本不可用，管线需要强制绝对 URL 并检查 HEAD 请求。
- 一些平台不支持脚注和任务列表，要提前在规范层禁止，而不是发布时降级。

## 可复用建议

把平台差异配置化，例如：

```yaml
newline: br | two_space | backslash
table: gfm | html | image
code_highlight: native | inline_style | image
footnote: true | false
max_table_cols: 4
image_base_url: https://...
```

这样 adapter 可以根据配置切换行为，平台策略不用散落在代码里。

如果做成 MCP，可以暴露三个工具：`markdown_lint`、`markdown_convert(target)`、`markdown_preview(target)`。Agent 生成完内容后必须调用 `markdown_lint`，通过后再 convert 到目标平台，preview 产出 human-in-the-loop 的确认点。不要追求全自动跳过确认，至少保留一个预览/人工发布 gate。

原始 Markdown 是唯一 source of truth，任何平台产物都可以由源码重建。发布记录保存 adapter 版本和 target，便于回滚审计。

## 总结

多平台发布的核心不是找一个大而全的转换库，而是先把问题收窄到可维护子集，再用 AST 校验、adapter 隔离平台差异、golden cases 守住回归。OpenClaw 场景中，Agent 负责快速生成，Markdown 管线负责让内容在不同平台稳定落地。宁可先支持 2-3 个平台做透，也不要试图第一天覆盖所有平台。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/c8d97d3d5b996518.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/fd51352b5542c6af.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/86bfb7a5b6156258.png)

