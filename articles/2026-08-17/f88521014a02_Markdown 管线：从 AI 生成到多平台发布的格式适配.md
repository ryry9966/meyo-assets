---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 33636
source: 综合讨论
publishedAt: 2026-08-17
---

## 背景

在 OpenClaw 的 Agent 编排里，让模型生成一篇 Markdown 并不难。真正的麻烦发生在发布环节：同一篇内容要发公众号、掘金、GitHub、Notion 或 Obsidian，每个平台对 Markdown 的支持都不一样。

如果让 Agent 直接面向某个平台写作，很快会陷入三件事：生成格式脏、平台规则混在 prompt 里难以维护、换个平台就要重写一遍。更实际的问题是，AI 输出的 Markdown 经常“看起来对”，一进平台就变形。

## 问题

典型的 AI 生成 Markdown 会有这些坑：

- 标题层级跳级，一会儿 `#` 一会儿 `###`
- 代码块缺少语言标识，或使用 `js`、`py` 这类平台不认识的别名
- 表格列数不一致，移动端直接溢出
- 本地图片路径、base64 图片、临时图床链接混杂
- frontmatter 被平台当成正文渲染
- 脚注、数学公式、原始 HTML 在部分平台直接丢失或显示源码
- 嵌套列表过深，公众号等富文本平台排版崩坏

平台差异也很大：GitHub 对 GFM 支持好，公众号本质上是内联样式 HTML，Notion 需要 blocks 结构，Obsidian 对 wikilink 和 embed 有自己的解析方式。

所以核心矛盾不是“转换器不强大”，而是源格式不可控、适配规则散落、结果无法校验。

## 做法 / 步骤

我把这条管线拆成四层：`normalize -> adapt -> validate -> publish`。

### 1. 中间格式规范化

先不直接做平台适配，而是把 AI 输出统一成一份中间 Markdown，存 Git 作为唯一事实源。

规范子集建议尽量保守：

- 只用一级到三级标题，且每个节点只出现一次一级标题
- 代码块必须有语言标识，并统一到 `javascript`、`python`、`bash` 这类全名
- 表格列数一致，单行不超宽
- 图片链接只允许远程 URL，禁止 base64 和本地相对路径
- 不使用脚注、原始 HTML、数学公式，除非目标平台明确支持
- 剥离 frontmatter，只保留正文

这一步可以用 remark/unified 之类的 AST 工具做，也可以直接通过 OpenClaw 的 Agent 后处理节点完成。关键是：**规范化后的 Markdown 必须能通过同一套 lint 规则**。

### 2. 平台 adapter 转换

每个平台一个 adapter，输入是规范化后的 Markdown，输出是该平台需要的格式。

例如：

- 公众号 adapter：解析 AST，代码块转成带内联样式的 `<pre>`，表格转 HTML，图片上传到微信素材，嵌套列表压平到两级以内
- GitHub / 掘金 adapter：保留 GFM，做代码语言别名映射，删除平台不支持的扩展语法
- Notion adapter：按 blocks 结构切分，标题、段落、代码块、表格分别映射到 Notion blocks API
- Obsidian adapter：将标准 Markdown 链接转成适合 vault 的 wikilink，必要时补 embed 语法

adapter 不要写成一个大函数，而是无状态、可独立测试的小模块。在 OpenClaw 里可以包装成插件或 MCP server，方便 Agent 按目标平台调用。

### 3. 发布前校验

每个 adapter 输出后加一个 validate 步骤，不通过就不发布。常见校验项：

- 目标是公众号：检查标题长度、代码块是否已转 HTML、图片是否已上传、表格是否转 HTML
- 目标是 Notion：检查 blocks 是否超过单次 API 限制，表格是否拆成合法结构
- 通用：检查正文是否残留 frontmatter、本地图片路径、`<script>` 等原始 HTML

校验可以用 dry-run 输出转换产物，再和源 Markdown 做 diff 确认。

### 4. 发布与回执

发布层记录每次发布用的 markdown hash、adapter 版本、目标平台、发布状态。API 失败要支持重试和幂等，尤其是公众号草稿箱和 Notion 这类有频率限制的平台。

## 踩坑点

1. **不要跳过一个中间规范层。** 直接让 AI 输出平台风格是最容易失控的做法。
2. **代码块语言别名不能靠猜。** `js` 在部分平台不识别，adapter 里要维护一张映射表。
3. **表格是重灾区。** 移动端宽度有限，公众号里复杂表格最好转成图片，或拆成列表。
4. **图片一定要处理成远程 URL。** 本地路径在发布端必然是死链；base64 内容平台 API 多半不收。
5. **数学公式不要硬塞。** 除非目标平台支持，否则宁可做成图片或代码片段。
6. **frontmatter 很容易残留。** 有些 Markdown 解析器会把它当正文，adapter 必须显式剥离。
7. **平台 API 不是稳定层。** 公众号、Notion 的字段和限制经常变，adapter 和校验规则要跟着维护。

## 可复用建议

- 源 Markdown 存 Git，平台产物作为构建结果，不要手动修改平台端。
- 每个 adapter 配套一组 golden case：10–20 条固定样例，跑一遍看输出是否稳定。
- 在生成 prompt 里尽早约束 AI：只准用二级标题、代码块必须标注语言、禁止脚注和 HTML、禁止 emoji 装饰。这比事后清洗划算。
- 把 adapter 做成 OpenClaw 插件或 MCP 工具，方便在不同 Agent 工作流里复用。
- 保留转换前后的 diff，问题回溯时能看到是源格式问题还是 adapter 问题。

## 总结

Markdown 多平台发布的核心不是做一个“万能转换器”，而是把工作拆成三层：源头约束、中间规范、adapter 适配。这样 AI 生成端只需保证进入规范子集，平台差异全部交给可测试的 adapter 处理。

在 OpenClaw 这类 Agent 自动化环境里，这套管线尤其适合落地：Agent 负责生成和调度，插件/MCP 负责转换，校验不通过就终止发布。最终效果是同一份 Markdown 源，稳定输出到多个平台，而不是每个平台各维护一份越来越脏的内容。

---

