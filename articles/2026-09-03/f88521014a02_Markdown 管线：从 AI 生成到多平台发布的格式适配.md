---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 35896
source: 综合讨论
publishedAt: 2026-09-03
---

## 背景

在社区里做内容自动化的同学大多走过类似的路：让 Agent 生成 Markdown，再手动贴到公众号、知乎、博客。写一篇还好，写一个系列就会发现时间全耗在“格式搬运”上——公众号不吃 Markdown，知乎会吞表格，静态站又要 frontmatter。我们的做法是把格式适配从 Agent 手里拿走，做成一条确定性的 Markdown 管线。

## 问题

直接让模型输出各平台格式，有三个明显缺陷：

1. **不可复现**。同一个 prompt 跑两次，内联样式和结构细节都不一样，排版变成抽卡。
2. **污染内容源**。模型为了适配平台会在正文里塞 HTML，源文件不再是干净的 Markdown，后续没法 lint、没法批量改版。
3. **平台规则在变**。适配逻辑散落在 prompt 里，规则一改就得重写提示词，而不是改代码。

## 做法

核心原则一句话：**Agent 只负责内容，格式转换交给确定性代码**。管线分四层。

1. **规范层**：约定唯一的 canonical Markdown。frontmatter 固定字段（title/tags/summary/targets），标题从 h1 开始，禁用内联 HTML。跑 markdownlint 卡住不一致的生成物。
2. **归一化层**：一个清洗脚本处理共性问题——去掉模型爱加的冗余空行和重复强调、外链图片转存图床并重写 URL、mermaid 预渲染成图片。
3. **适配层**：每个目标平台一个 adapter，在 OpenClaw 侧注册成独立 MCP tool 或 CLI 子命令。公众号走 markdown-it 转 HTML 后用 juice 内联样式；知乎输出降级版 Markdown（宽表格转图片或直接删减）；静态站保留 frontmatter 原样直出。
4. **校验层**：dry-run 渲染各目标产物存 snapshot，diff 暴露变化。发布前人眼过一遍预览页。

Agent 侧只暴露一个统一的 publish 工具，参数就是目标平台列表，降级和转换全部封在工具内部。

## 踩坑点

- 公众号会剥掉 class 和外部样式表，所有样式必须内联；`<pre><code>` 在 iOS 上会被压行，代码块要限行宽、逐行包 span 上色。
- 知乎会吞 footnote 和 task list，别指望优雅降级，发布前直接删。
- 图床防盗链是隐形炸弹：本地预览一切正常，发布后图片全挂。公众号有素材库机制，外链图必须重新上传。
- frontmatter 忘记剥离，正文开头多一段 YAML，读者一脸问号——校验层就是为这种事存在的。
- 模型生成的标题层级经常从 h2 开始，一条 lint 规则比在 prompt 里反复叮嘱有效得多。

## 可复用建议

- canonical source 只有一份；adapter 只做减法和改写，不做内容增删。
- 适配逻辑进代码库跟随版本管理，prompt 里只留“输出规范 Markdown”这一条约束。
- 每个平台 adapter 配一份最小样例文档跑进 CI，平台悄悄改规则时 diff 立刻报警。
- 新增平台时先写“该平台不支持什么”清单，降级规则从清单推导，不靠猜。

## 总结

这条管线没有任何黑科技，价值在于把“格式”从概率性的模型输出里剥离出来：Agent 产出规范的 Markdown，代码负责全部平台差异，校验兜底。改一个平台规则只动一个 adapter，内容源永不搬家。当内容生产从一篇变成一百篇时，这套东西就是你的护城河。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/0dcc953620e79eb2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/a763febcdd6c88fd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/1907356709325d8f.png)

