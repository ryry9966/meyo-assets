---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 31737
source: 综合讨论
publishedAt: 2026-08-05
---

## 背景

在 OpenClaw 社区，我们经常让 Agent 直接产出 Markdown 草稿，然后手动复制到公众号、知乎、掘金、博客园等平台。问题在于：这些平台的 Markdown 渲染器“方言”各异，同样的源码会呈现出完全不同的排版。代码块主题丢失、表格挤压变形、图片路径失效、脚注语法报错……这导致每次发布前都要手工处理一遍格式，自动化链路易碎且难以复用。

## 问题

核心矛盾有三个：

1. **方言差异**：GitHub Flavored Markdown（GFM）中的表格、任务列表、自动链接，在 CommonMark 或平台私有解析器下并不通用。
2. **平台限制**：公众号不支持表格，知乎不支持 Mermaid 图表，掘金不解析脚注，CSDN 对 HTML 标签过滤严格。
3. **样式缺失**：平台对 `class` 属性、外链、图片尺寸的处理方式不同，同一个 `<pre>` 块在不同地方显示效果差异巨大。

如果让 AI 直接输出“最终 HTML”，又会失去 Markdown 作为单一事实源的可维护性。因此，我们需要的是一条“源 Markdown + 平台适配器”的管线。

## 做法

我的做法分四层，全部可以嵌入 OpenClaw 的 MCP 工具链。

### 1. 限定源 Markdown 子集

建立 `markdownlint` 配置，只允许 CommonMark 加上 GFM 的表格和任务列表（因为这两个被多数平台接受）。禁用脚注、Mermaid 代码块、数学公式（源码中允许，但转换时提示需替换为图片）。在 Agent 的 system prompt 中写入这份规则，并在生成后强制校验。

### 2. 生成后校验

在 OpenClaw 中注册一个 `md_check` MCP 工具，调用 `markdownlint-cli2` 和自定义正则，检查：

- 标题层级是否连续（不允许 h2 → h4 跳跃）
- 表格是否含复杂合并或单元格内换行
- 图片路径是否为绝对 URL
- 是否存在非白名单语法

校验失败则直接要求 Agent 修正，而不是进入下一环节。

### 3. 统一转换

使用 Pandoc（`f:gfm` → `t:html`）作为主转换器，但不用默认模板。我写了一个 Lua 过滤器，专门处理代码块高亮 class、表格包裹布局、链接 target。例如：

- 为公众号生成 `style="border-collapse:collapse"` 的内联表格样式
- 为知乎生成 `figure` 包裹的代码块容器
- 对掘金移除 `id`/footnote 相关标记

### 4. 平台后处理脚本

针对强特殊需求写小脚本。例如，微信不渲染任何表格，我写了一个 Python 脚本：

- 用 Playwright 把 Pandoc 输出的 `<table>` 截图
- 把图片上传到自己的图床
- 用 `<img>` 替换原表格位置

同样，图片相对路径统一替换为 CDN 绝对地址，视频链接替换为平台支持的卡片语法。

最后在 OpenClaw 中把以上步骤封装成 `publish_doc` MCP 工具，输入 `md_path` 和 `platform`，返回转换后的 HTML 或可直接复制到编辑器剪贴板的内容。

## 踩坑点

- **Pandoc 的表格解析并不宽松**。GFM 的复杂表格（合并单元格、单元格内换行）会直接报错，必须在校验阶段就过滤掉。简单表格没问题，但 GitHub 生成的某些表格用了多层嵌套，要提前清理。
- **公众号的“内联样式”是假内联**。微信官方编辑器会过滤掉大部分 `class`，只保留 `<span style="...">` 里的基础 CSS，但 `nested span` 多次嵌套时也会丢样式。最好把高亮颜色直接写在最内层元素的 `style` 上。
- **Agent 经常忽略 markdownlint 的警告**。尤其“标题层级跳跃”这类规则，AI 生成的文档几乎必然违反。需要把校验结果作为硬失败（返回非零），而不是“提示”，否则 Agent 会跳过。
- **脚注和参考链接容易混淆**。很多平台支持 `[text][ref]`，但不支持 `[^1]`，而 AI 生成时两种混用。统一在源子集中禁用脚注，需要时用参考链接 + 末尾手动列表代替。
- **平台 API 不如剪贴板可靠**。CSDN 的博客 API 对 Markdown 字段有长度和标签限制，不如用 `pbcopy` 将转换后的 HTML 直接粘贴到编辑器中。

## 可复用建议

维护一个 `platform-pipeline/` 项目目录：

```
pandoc/            # Lua 过滤器，按平台命名
scripts/           # convert.py, table2img.py
tests/             # sample.md, 各平台渲染快照
config/            # markdownlint.yml, platform.json
```

在 `platform.json` 中声明每个平台启用的语法特性和后处理规则。每次改动转换器后，在 CI 中对 `sample.md`（包含表格、代码块、图片、列表、引用五种元素）跑一遍所有平台输出，做 diff 回归。

我强烈建议把可接受的 Markdown 子集写死在 AI 的 system prompt 里，而不是事后靠 lint 修复。先“限制”，再“适配”，比“万能解析”更务实。

另外，`publish_doc` 这类工具在 OpenClaw 中很适合做成通用 MCP server，这样任何 Agent 都能直接调用，不必重复造轮子。

## 总结

从 AI 生成到多平台发布，不只是一个“复制粘贴”动作，而是一个边界明确的工程问题。通过限定源子集、统一转换、平台后处理这三个层次，可以显著减少手动调整，让 Agent 真正跑通“写稿 → 校验 → 转换 → 发布”的闭环。在 OpenClaw 社区，这条管线可以作为 MCP 插件直接共享，值得继续迭代。

---

