---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 37143
source: 综合讨论
publishedAt: 2026-09-12
---

## 背景

在 OpenClaw 的典型工作流里，Agent 的写作产出几乎都是 Markdown：MCP 工具返回的报告、定时任务生成的周报、插件产出的文档。但"写成 Markdown"和"发出去能看"之间隔着一条鸿沟——公众号要全内联样式的 HTML，知乎吃自家编辑器那套，静态站要 front matter，掘金、Dev.to 又各有一套方言。多数人的做法是每个平台手工调一遍格式，内容一更新，全部重来。

## 问题

拆开看，真正的痛点有三个：

1. **LLM 输出不可控**。模型随手堆三四级标题、裸 HTML 标签、不规范的表格，下游渲染直接崩。
2. **平台方言不一致**。CommonMark、GFM、各家私有扩展，同一份源文件渲染结果差异很大。
3. **发布不可重放**。改个错别字就要全平台重新粘贴，没有幂等性。

## 做法

我的思路是把 Markdown 当成中间表示（IR），发布当成编译，每个平台一个后端：

**第一步：收敛源头。** 在 Agent 的 system prompt 和输出 schema 里明确允许的 Markdown 子集：只用 H2/H3、标准列表、标准表格、fenced code block，禁止裸 HTML 和 footnote。源头约束比事后修复便宜十倍。

**第二步：规范化层。** AI 输出先过 unified/remark 管线：markdownlint 校验、标题统一降级（H1 一律降为 H2）、图片链接转存图床、front matter 填充元数据。这一层的产出是"标准 Markdown"，之后所有平台只消费它。

**第三步：平台适配器。** 每个适配器是纯函数：`render(markdown, theme) → output`。公众号适配器把 Markdown 转成全内联样式的 HTML（公众号会剥掉 `<style>` 标签），主题是 JSON 配置的样式 token；静态站适配器直接透传标准 Markdown 加 front matter；知乎走 API 或剪贴板富文本。

**第四步：发布工具化。** 每个适配器包成 MCP tool 或插件命令，发布前对内容做 hash，hash 未变就跳过，实现幂等。

## 踩坑点

- **公众号外链**：正文外链不可点。我的处理是在规范化层自动转成文末参考列表（编号 + 纯文本 URL）。
- **表格溢出**：移动端列多必炸，规范层限制最多 3 列，超出的自动转列表。
- **嵌套列表缩进**：2 空格和 4 空格在不同渲染器表现不同，remark 统一成 4 空格。
- **代码高亮**：公众号需要把 highlight.js 输出转成带内联颜色的 span，直接贴 CSS 类是无效的。
- **图片防盗链**：内容发到知乎后图片 403，管线里加一步图片转存再改写 URL，别依赖源站。

## 可复用建议

1. 先写一页"格式契约"文档，明确允许的 Markdown 子集，给 Agent 和人都看。
2. 每个适配器配 golden file 快照测试，防止样式 token 改动引发回归。
3. 主题与内容分离，全部样式 token 收进一个 JSON。
4. 别追求覆盖所有语法，子集越小，管线越稳。

## 总结

Markdown 管线的本质是"一次生成、多端编译"。把约束放在源头，把差异关进适配器，把发布做成幂等工具，AI 的产出才能真正一键分发，而不是每换一个平台就重新劳动一次。这套思路不绑定任何具体框架，用 remark、markdown-it 或者自己写解析都成立，核心是分层和纯函数。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/1e14d9735c9f12da.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/f0d6dd63a8769ad2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/07707bec36b703a2.png)

