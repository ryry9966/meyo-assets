---
title: Markdown 管线：从 Agent 输出到多平台发布的可复用适配
feedId: 34343
source: 综合讨论
publishedAt: 2026-08-23
---

## 背景

在 OpenClaw 的自动化内容场景里，很多 Agent 默认输出 Markdown。问题在于，发布目标不是同一个编辑器，而是公众号、Notion、Telegram 频道、X/微博等完全不同的平台。直接复制粘贴，要么格式丢失，要么样式错乱，要么外链和图片全部失效。

如果每个 workflow 都写一套转换逻辑，很快就会变成难以维护的 if-else 堆叠。更稳定的做法是建一条短管线：把 Agent 输出固定为单一 Markdown 源，再做规范化，然后由适配器输出各平台需要的格式。

## 问题

AI 生成的 Markdown 通常不算干净：

- 文件头部残留 frontmatter，发布端不识别；
- 一段话被拆成多行，或者多余空行过多；
- 嵌套列表层级不准，第三层经常丢；
- 代码块内外特殊字符没有妥善处理；
- 图片使用的是临时 URL，换平台后失效。

平台差异也很大。公众号读的是 HTML，且外链和图片受限制；Notion 需要 block 结构；Telegram 只支持有限的 HTML 标签；X 基本只能当纯文本处理。如果让 Agent 针对每个平台改 prompt，输出不稳定；如果让每个平台各自解析原始 Markdown，格式一定漂移。

## 做法 / 步骤

### 1. 固定单一事实源

所有内容先落盘为 `canonical.md`。统一 frontmatter，例如：

```yaml
---
title: 标题
tags: [agent, automation]
summary: 一段摘要
cover: https://cdn.example.com/cover.png
platforms: [wechat, notion, telegram, x]
---
```

平台信息放在 frontmatter 里，不在正文中散落判断逻辑。

### 2. 做一次规范化

写一个 `normalize` 脚本，只做内容清洗，不做平台判断：

- 剥离未知 frontmatter 字段；
- 统一换行为 LF；
- 压缩三个以上的连续空行；
- 修复未闭合的代码块围栏；
- 将常见 HTML 标签，比如 br，转换为 Markdown 换行；
- 统计标题层级，确保从 h2 或 h3 开始，避免一篇内容里出现多个 h1。

这一步的关键是幂等。规范化后的 Markdown 应该可以作为后续所有平台适配的输入。

### 3. 适配器模式

每个平台一个 adapter，暴露统一接口：

```text
transform(content: string, platform: Platform, options?: Options): string
```

公众号 adapter 用 markdown-it 把 Markdown 转成 HTML，并注入内联样式：p 段距、code 背景、h2 边距。公众号不支持外部 CSS，所以样式必须内联。

Notion adapter 把 Markdown 解析成 blocks，注意嵌套列表和代码块。不要直接使用现成的 md-to-notion 包处理三层以上列表，容易丢层级。可以按缩进计数手工构建 block 树。

Telegram adapter 使用 HTML parse mode，只保留 Telegram 支持的标签。复杂表格和嵌套引用降级为 monospace 文本。

X adapter 提取 title 和前若干字，拼上链接。它不做复杂格式保留，只保证文字通顺和关键信息完整。

### 4. 产物落盘，而不是直接发布

统一输出到 `dist/`：

```text
dist/wechat.html
dist/notion.json
dist/telegram.html
dist/x.txt
```

这样每次改 adapter 或 parser，都能用 git diff 看清楚每个平台的具体变化，也方便回归排查。

### 5. 发布 action / MCP 工具

可以把上述流程封装成 CLI 脚本，也可以做成 MCP server。暴露两个工具：

- `markdown_normalize`：输入原始 Markdown，输出规范化结果；
- `markdown_publish`：输入 `canonical.md` 和 platforms 数组，输出各平台 payload 或直接触发发布。

在 OpenClaw 的 workflow 里，发布节点只调用这两个工具，不直接接触平台细节。

## 踩坑点

- **公众号代码块**：外部 CSS 不生效，必须在生成 HTML 时对 pre/code 写内联样式，并转义代码内的 `<`、`>`、`&`。
- **公众号图片**：外链图片不会自动上传，需要在 adapter 里替换为已上传素材 URL，否则图片裂开。
- **Notion 嵌套列表**：很多 md-to-notion 工具会丢失第三层。建议在 adapter 里根据缩进手工构建 block 树，不要盲信第三方解析器。
- **Notion 请求限长**：一次 append 太多 blocks 会被截断。按 100 个 blocks 分批提交比较稳妥。
- **Telegram 标签限制**：不支持嵌套 blockquote 和复杂表格。遇到复杂结构时，降级为普通文本或省略，而不是强行保留。
- **X 链接占字数**：链接会被计入长度，建议用短链，或只发摘要和原文入口。
- **frontmatter 误判**：只在文件头部连续的 `---` 之间解析 frontmatter，避免把正文里的分隔线当元数据。

## 可复用建议

- **平台差异配置化**：不要在 normalize 里写平台判断，normalize 只做清洗；平台逻辑全部放在 adapter。
- **建立 golden test**：准备固定样例，为每个平台保存期望输出。改 parser 或 adapter 后跑测试，避免静默回归。
- **CI 检查**：用 markdownlint、prettier 检查源格式，同时检查 adapter 输出是否包含关键段落。
- **MCP server 拆分**：normalize 和 publish 分成两个工具。前者幂等，后者可重复执行，方便 agent 单独调用。
- **降级优先**：对纯文本平台，先保证文字通顺和关键信息不丢，再考虑格式。不要为了保留样式牺牲可读性。

## 总结

稳定发布不靠 prompt 碰运气，而靠一条短管线：canonical Markdown → normalize → adapter → platform output。先把源格式固定，再用适配器隔离平台差异，最后用测试防回归。这样 Agent 生成的内容可以从一份 Markdown 变成多端产物，减少返工，也方便后续接新平台。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/ee2dc4efe01418ee.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/b94716564d481cf0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/8d9e8c69c043be07.png)

