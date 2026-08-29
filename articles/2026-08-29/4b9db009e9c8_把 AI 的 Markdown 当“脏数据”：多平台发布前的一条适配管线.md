---
title: 把 AI 的 Markdown 当“脏数据”：多平台发布前的一条适配管线
feedId: 35212
source: 综合讨论
publishedAt: 2026-08-29
---

## 背景

在 OpenClaw / Agent 自动化发布流程里，我们经常让模型直接生成 Markdown，然后推到公众号、知乎、掘金、Notion 或自己的静态站。问题在于：模型输出是自然语言生成的“类 Markdown”，不是稳定的数据格式。今天多一个空行，明天代码块少了语言标注，后天 frontmatter 没闭合。平台渲染差异再叠加一层，排版事故就会频繁出现。

## 问题

- **LLM 输出风格漂移**：同一个 prompt 下，有时用 ```` ```python ````，有时不写语言；标题层级偶尔跳过；列表嵌套时好时坏。
- **平台方言不兼容**：公众号需要内联样式 HTML，知乎/掘金对 HTML 白名单不同，Notion/Obsidian 对 GFM 的支持也不一致。
- **重试无效**：格式问题不是语义问题，让 Agent 重写经常引入新问题，反而更不可控。

## 做法：一条确定性管线

我目前的做法是让 AI 的原始 Markdown 先经过一段规范化管线，再进入各平台适配器。核心原则：**不信任 LLM 输出，只信任 AST**。

### 1. 固定输出契约

在 OpenClaw 的 prompt / tool 定义中，要求模型只输出：

```yaml
---
platforms: [wechat, zhihu, juejin]
slug: example-post
summary: ...
---
正文（GFM Markdown）
```

插件侧先做 frontmatter schema 校验，不通过直接让 agent 重新生成，而不是试图修复一坨已经错乱的文本。

### 2. 解析为 AST，而非正则

使用 unified / remark-parse 解析，开启 gfm 和 frontmatter 插件。后续操作全部基于 AST 节点，避免正则拆块。嵌套列表、引用里的代码块，正则很容易拆坏，AST 则相对可靠。

### 3. 规范化

一些固定规则：

- 配平代码围栏：统计 ```` ``` ```` 是否成对，奇数则在文末补一个。
- 统一语言标签：`javascript -> js`、`shell -> bash` 等，避免高亮失效。
- 清理未授权 HTML 标签：`script`、`iframe`、`style` 直接丢弃。
- 裸 URL 转为链接。
- 表格列数不一致时，降级为列表。
- 图片地址转本地图床，补齐 alt。

### 4. 平台适配器

以 AST JSON 作为中间表示，各平台适配器只做渲染差异，不做内容修复：

- **公众号/头条**：remark-rehype 转 HTML，给 `pre`、`code`、`table` 注入内联样式，脚注转括号注。
- **知乎/掘金**：保留 Markdown，但过滤 `<details>`、脚注，转义平台不支持的 HTML。
- **Notion/Obsidian**：保留 GFM，但把 Callout 转成对应语法，frontmatter 格式对齐。

### 5. Dry-run 校验

每次发布前生成目标产物，并和源 AST 做节点数对比。比如源有 12 个 blockquote，目标也要有 12 个；源有 3 个 code block，目标只能多不能少。偏差超过阈值就拦截发布。

## 踩坑点

- **正则拆块不可靠**：嵌套列表、引用里的代码块都会翻车。用 AST 后这类问题基本消失。
- **模型会吞 frontmatter 结束符**：必须校验 YAML 是否完整。如果正文开头被当成 frontmatter，后续节点会全丢。
- **公众号表格溢出**：列太多或包含长代码时，前端渲染很惨。适配器里限制列数，或者拆成多表。
- **外链图片防盗链**：自动化发布后图片全挂，常见却容易忽略。管线里应默认下载到本地或图床，并返回稳定 URL。
- **平台草稿接口差异**：有些平台需要 HTML，有些要 Markdown，甚至同一平台不同接口不同。适配器最好按“输出格式”而非“平台名”拆分，例如 `html-styler`、`md-stripper`、`notion-importer`。
- **依赖升级会导致输出微妙变化**：unified / remark 生态版本变化可能改变空白符。建议锁版本，并保存 golden 样本做回归。

## 可复用建议

- 把管线拆成三个独立模块：`normalizer`、`adapter`、`publisher`。OpenClaw 里可以封装成 MCP 插件，输入 raw markdown，输出各平台产物路径。
- 保留 canonical 源文件，平台产物放 `dist/{platform}/`。永远不要直接改模型输出。
- 给每个 adapter 写几个黄金用例：标题、嵌套列表、代码块、表格、图片。升级依赖后跑一遍 diff。
- 在日志里记录每个 transform 处理的节点数量，发布失败时能快速定位是规范化丢内容还是适配器丢内容。

## 总结

AI 生成 Markdown 到多平台发布，真正的问题不是“写得不够好”，而是“格式不可控”。把 LLM 输出当作脏数据，用 AST 做规范化，再用适配器隔离平台差异，能显著减少排版事故。这套管线不复杂，但一旦落地，后面接新平台只是多一个 adapter 的事。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/3de950fdf421ec09.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/6cff729dd954367a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/011078e5bce2bfe8.png)

