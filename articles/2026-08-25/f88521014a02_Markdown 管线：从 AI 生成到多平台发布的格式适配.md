---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 34625
source: 综合讨论
publishedAt: 2026-08-25
---

## 背景

在 OpenClaw、Agent、MCP 这类自动化内容管线中，Markdown 几乎是 AI 生成文本的事实标准。它可读、可版本化、解析器生态成熟，非常适合作为中间格式。可一旦内容要分发到公众号、知乎、掘金、Notion、飞书、WordPress 等不同平台，问题就集中爆发：各平台对 Markdown 的支持程度、扩展语法、渲染策略都不一样。同一份 Markdown 直接复制过去，轻则样式丢失，重则代码块、表格、公式全部乱掉。

更麻烦的是，AI 生成的内容经常带有平台不接受的语法：内嵌 HTML、非标准 emoji、过深标题层级、无语言标注的代码块、脚注、base64 图片等。如果只在发布前做一次性转换，出错后很难定位是生成端还是转换端的问题。

## 问题本质

多平台发布的核心矛盾是：Markdown 是“语义格式”，而发布平台需要的是“渲染结果”。平台之间对 Markdown 的解析差异，本质上是对同一语义的不同渲染实现。要解决这个问题，不能只靠正则替换，更不能靠人工逐个平台调试。

## 做法/步骤

### 1. 约束 AI 输出为可移植 Markdown 子集

在上游就减少不确定性。在 prompt 或 Agent 工具描述里明确要求：

- 禁止内嵌 HTML、内嵌样式、脚注、base64 图片
- 标题层级不超过 H3
- 代码块必须带语言标注，例如 ```python
- 列表统一用 `-`，不用 `*` 或 `+`
- 分割线统一用 `***`，避免与 YAML front matter 混淆
- 表格列数不超过 4 列，复杂表格改为列表描述

这样能过滤掉 80% 的平台兼容问题。

### 2. 建立平台 profile

每个目标平台维护一份配置，定义支持/不支持的语法、替换规则、图片处理策略、换行策略、代码块样式。例如：

- 公众号：不支持原生 Markdown，需转成内联样式 HTML，代码块要转成公众号可接受的 `<pre>` 样式
- 知乎：支持部分 LaTeX，但复杂表格会溢出，需要转换为图片或列表
- Notion：导入 Markdown 时会丢失部分 GFM 语法，需要提前降级处理
- 掘金：支持 front matter，但对 HTML 标签过滤较严

这份 profile 就是转换器的“规则来源”。

### 3. 基于 AST 转换，不要用正则

用 `remark` 或 `markdown-it` 先把 Markdown 解析为 AST，再按平台 profile 做插件式转换。AST 方案可以精确保留代码块语言、表格结构、列表层级，而正则在这些地方极易出错。

例如转换公众号 HTML 时，针对 `code` 节点提取语言并生成带背景色的 `<pre>` 段；对 `table` 节点检查列数，超过 4 列时自动降级为列表。

### 4. 发布前验证

发布前用 headless 浏览器或平台预览 API 做渲染检查，至少覆盖：

- 代码块是否保留语言标注和缩进
- 表格在移动端是否溢出
- 图片链接是否可达，是否防盗链
- 数学公式是否被平台渲染或正确降级
- 标题层级是否被平台重新编号

可做一个轻量验证器，检查 AST 节点数量、代码块语言分布、图片 URL 状态码。

### 5. 封装为 MCP 工具或 OpenClaw 插件

把转换逻辑封装成可复用工具，例如：

```text
convert_markdown(source, profile) -> {text, html, assets}
validate_rendered(html, profile) -> {passed, issues}
publish_content(text, platform, credentials) -> {url}
```

这样 Agent 可以按需调用，不必在每个流程中重复实现转换逻辑。

## 踩坑点

- **单换行处理**：有的平台把单换行当空格，有的当 `<br>`。预处理时统一行尾加两空格或显式 `<br>`，但有些平台会过滤。
- **代码块语言丢失**：正则替换容易把 ` ```python ` 变成纯文本。必须基于 AST 提取语言并映射到平台样式。
- **表格溢出**：移动端表格列数一多就横向滚动甚至截断。建议列数不超过 4，否则改写成列表。
- **图片外链**：部分平台会防盗链或拒绝对外链图片的加载。建议把图片下载后上传到平台图床或对象存储，再替换链接。AI 生成的占位图 URL 尤其容易出现 403。
- **front matter 混淆**：AI 有时输出 `---` 当作分割线，但发布平台可能解析为 YAML front matter 或直接忽略。统一改用 `***`。
- **HTML 标签被过滤**：公众号会过滤 `<style>`、`<script>` 和大部分 class。不要在 Markdown 中依赖 HTML 实现样式。
- **数学公式**：知乎支持部分 LaTeX，公众号完全不支持。对不支持平台，公式应转成图片或纯文本。

## 可复用建议

- 保留原始 Markdown 作为 source of truth，任何转换产物都可以重新生成。
- 平台差异做成配置表，不要散落在代码分支里。
- 转换逻辑用 AST + 插件机制，不要写一长串正则。
- 在自动化流程中加入渲染验证步骤，避免发布后才发现问题。
- 对 AI 输出增加格式校验器，不符合子集的自动修正或要求重新生成。
- 转换工具做成 MCP server，让 OpenClaw Agent 可调用 `convert`、`validate`、`publish` 三个工具。

## 总结

Markdown 管线不是简单的格式转换，而是把 AI 生成内容约束到可预测子集，再按平台渲染差异做确定性适配。真正有用的部分在于上游约束和发布前验证，而不是转换器本身。把规则配置化、转换 AST 化、验证自动化，多平台发布才能稳定运行。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/eeea8e10d44415b6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/544b27b503c91932.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/ec4de70df1adc560.png)

