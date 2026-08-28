---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 35115
source: 综合讨论
publishedAt: 2026-08-29
---

在 OpenClaw 的 Agent 工作流里，AI 生成 markdown 已经不是难事。但把生成结果发布到公众号、知乎、掘金、个人博客或 Notion 时，经常会出现格式错乱：代码块丢失高亮、表格被压扁、任务列表消失、外链被吞。这类问题不是单靠调提示词能完全解决的，需要在 AI 输出与平台渲染之间加一层“格式适配”。

问题主要集中在两点。一是 AI 输出不规范：常见的是整篇文章被包在 ````markdown` 代码围栏里，或者代码块语言标签写成 `Python`、`js ` 带空格，甚至混入 HTML 注释和未闭合标签。二是平台方言不一致：微信公众号基本是受限 HTML，不支持原生 GFM 表格、任务列表、脚注；知乎和掘金对 GFM 支持较好，但细节仍有差异；Notion 需要块级结构，API 导入时对嵌套列表和分隔线有要求；自建静态博客则依赖具体静态站点生成器。

我建议的管线分五步。

第一步，规范化 AI 输出。在 Agent prompt 里明确要求只返回正文 markdown，不要外层围栏，不要 HTML，代码语言标签使用小写。如果走 MCP 工具调用，可以让模型把文章放入 JSON 字段，减少 markdown 污染。

第二步，定义单一事实源。使用 frontmatter + 正文作为一个文档对象。例如：

```yaml
---
title: "..."
slug: "..."
summary: "..."
tags: [ai, markdown]
cover: https://...
draft: false
---
```

然后用 remark 或 unified 解析成 AST，后续转换尽量基于 AST，而不是正则替换。字符串替换极易破坏代码块内部内容。

第三步，做基础清洗与校验。写一个校验脚本，处理外层围栏、语言标签规范化、多余空行、标题层级跳级、图片外链。用 markdownlint 或 prettier 作为基础检查，不要自己从零写 lint。

第四步，平台 profile。为每个目标平台维护一份配置文件，描述差异化规则。例如 `wechat.yaml` 里定义表格转图片或 HTML、脚注转括号、外链统一文末引用。不同平台只是数据不同，转换器保持同一套逻辑。

第五步，发布脚本。从源 markdown 读取，根据 profile 生成各平台版本，输出 diff 和预览。确认后批量发布或定时发布。

踩坑点：一是 AI 输出里的 ````markdown` 围栏很容易被忽略，发布后正文开头会出现 ``` 字符；二是公众号会吞掉大部分 HTML，代码块行号、task list 都不可用，需要预转换；三是表格在移动端溢出，不要直接粘贴，可转成图片或精简表格；四是不要用字符串 replace 处理链接和图片，容易破坏代码块；五是外部图片防盗链，最好统一转存到自己的图床并替换 URL。

可复用建议：保留原始 markdown 作为源，任何转换都生成新文件，不就地修改；平台配置数据化，避免在脚本里散落 if-else；把格式校验和转换做成 MCP 工具或插件，让 Agent 发布前自动调用；用快照测试覆盖各平台转换结果，防止某个平台样式回退；只处理自己实际发布的平台子集，不要试图做万能转换器，维护成本会失控。

总结来说，AI 生成 markdown 只是管线起点，多平台发布的质量取决于中间表示是否稳定、平台差异是否被显式管理。与其在 prompt 里反复强调格式，不如把清洗、校验、转换、预览、发布固化成流水线。这条管线一旦搭好，OpenClaw 里的 Agent 才能真正把内容自动分发出去，而不是每次都依赖人工救火。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/0445957e55cf5a37.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/7e314acdfae346f6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/7ed637b0be3333a3.png)

