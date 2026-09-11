---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 37145
source: 综合讨论
publishedAt: 2026-09-12
---

## 背景

现在内容生产链路里，AI 负责初稿已经很常见：Agent 产出一份 Markdown，人审一遍，然后分发到公众号、知乎、掘金、自建博客等多个平台。问题在于，Markdown 只是"看起来通用"，各平台渲染器能力差异很大，直接把 Agent 输出原样贴过去，翻车率不低。

## 问题

实践中反复出现几类故障：

- 微信公众号编辑器会剥掉 class 选择器，只认 inline style，通用样式全部失效；
- 知乎对 HTML 走白名单，表格、脚注经常被丢弃；
- 代码块高亮依赖语言标签，AI 输出里 `js` / `javascript` / `JavaScript` 混用，渲染结果不一致；
- 图片外链有防盗链，发到第三方平台直接裂图。

根因是：Agent 生成的是"一份 Markdown"，而发布端需要的是"N 份按平台约束裁剪过的格式"，中间缺一个适配层。

## 做法

我们的管线分四步，核心思路是**单一事实源 + 平台 profile**：

1. **规范化**。Agent 输出先过 remark 解析成 AST，同时用 markdownlint 收敛写法：统一列表缩进、强调符号、代码语言别名。front-matter 里声明标题、摘要、标签和目标平台列表。
2. **平台 profile**。每个目标平台一份约束清单，例如：公众号 → 禁表格、禁脚注、代码块降级为带背景色的 pre；知乎 → 表格转图片或删减；掘金/博客 → 全功能直出。profile 用 JSON 描述，Agent 生成内容时读这份 schema，从源头避开不支持的语法。
3. **AST 级转换**。用 unified/remark 插件按 profile 改写 AST（表格→图片、脚注→文末参考段落、图片 URL 替换为平台图床地址），再序列化输出。不要用正则改原文，嵌套结构的边界 case 会被正则打穿。
4. **预览与发布**。各平台产物先本地渲染截图，人工确认后由 MCP 工具或脚本调平台 API 发布。发布失败时保留原始 Markdown，产物随时可再生。

## 踩坑点

- **别高估 AI 的 Markdown 合规性**。嵌套列表缺缩进、块引用后缺空行这类问题肉眼难察觉，但 remark 解析后结构会变，lint 必须放在管线最前面。
- **公众号图片必须先传微信图床**，外链要么被过滤要么裂图，URL 替换要做成转换插件而不是手动操作。
- **表格是重灾区**，列多在任何平台都容易溢出；转图片要控字号，不如在生成端直接限制列数。
- **序列化不是无损的**，remark 重排软换行后 diff 全是变更，需要对比就固定 formatter 配置。

## 可复用建议

- 把"平台支持什么语法"写成数据（profile JSON），而不是存在某个人或某个脚本里；
- 格式判断一律基于 AST，正则只用于清洗纯文本字段；
- 转换函数保持纯函数，输入 AST 输出 AST，方便单测；
- canonical Markdown 是唯一编辑入口，平台产物都是构建产物，不要反向修改。

## 总结

这件事的本质不是"格式转换"，而是把发布约束前置到内容生产环节。Agent 能读 profile，就能在生成时少踩坑；管线能结构化处理 AST，就能在发布时少出意外。链路打通后，我们一次审核可稳定分发四个平台，格式类返工基本归零。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/e98fc3d5553872a6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/a58ada903858a450.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/da0752ad859dc756.png)

