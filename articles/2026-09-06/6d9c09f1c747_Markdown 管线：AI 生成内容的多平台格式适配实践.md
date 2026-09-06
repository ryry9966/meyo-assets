---
title: Markdown 管线：AI 生成内容的多平台格式适配实践
feedId: 36349
source: 综合讨论
publishedAt: 2026-09-06
---

## 背景

社区里做内容自动化的常见模式是：Agent（或直接调 LLM）产出 Markdown 初稿，然后发布到公众号、知乎、掘金、GitHub 仓库等多个平台。生成环节目前已经比较成熟，真正的工程量集中在下游——**格式适配**。

## 问题

各平台对 Markdown 的支持差异很大：公众号编辑器只认一小部分 HTML，且要求样式全部内联；知乎对部分扩展语法会静默降级；掘金和 GitHub 基本符合 GFM；外链图片在部分平台会被防盗链拦截。如果为每个平台手工改一遍，费时且容易漏。更麻烦的是，AI 生成的 Markdown 本身就有变体不稳定的毛病——列表缩进、表格对齐、代码块语言标注时对时错。

## 做法

核心思路：**单一事实源 + AST 级转换 + 平台 adapter**。

1. **约束生成。** 给 Agent 定一个“安全子集”规范：只用 CommonMark + GFM 基础语法，禁用裸 HTML、脚注、数学公式这类兼容性差的扩展；元数据统一放 YAML frontmatter。这份规范直接写进 system prompt 或输出模板。
2. **生成后先过 lint。** 用 markdownlint 做机械校验（缩进、空行、代码块闭合），不合规就退回模型重写或走规则修复，不要让脏输入进转换层。
3. **AST 规范化。** 用 remark/unified 解析成 AST，在这一层做通用清理：剥离 frontmatter、归一化图片引用、把过宽表格降级为列表。
4. **平台 adapter。** 每个目标平台一个小模块，输入同一棵 AST：
   - 公众号：rehype 转 HTML，代码块在构建时用语法高亮器着色成 inline style，图片转存微信素材库；
   - 知乎 / 掘金：序列化回 Markdown，只处理平台特有差异；
   - 自建博客：保留 frontmatter 直接产出。
5. **校验与发布。** 每个 adapter 的输出做 golden file 快照对比，diff 异常即中断；最后通过平台 API 或 CLI 推草稿，人工确认再发。

整条链可以封装成 MCP tool 暴露给 Agent，例如 `markdown.lint`、`markdown.transform(target)`、`publish.draft`，让“生成”和“适配发布”彻底解耦。

## 踩坑点

- **正则替换是陷阱。** 看着快的正则在嵌套列表和代码块里必翻车，一律在 AST 层操作。
- **公众号样式内联必须构建时完成**，运行时注入 `<style>` 会被剥掉。
- **外链图片是隐形炸弹**，知乎和公众号都可能挂掉。转存应做成管线的固定步骤，而不是发布失败后补救。
- **公众号正文不支持外链锚文本**，链接要转成文末引用列表。
- **AI 输出的 frontmatter 偶尔带注释或多余字段**，解析要宽容、校验要严格。

## 可复用建议

- 先写“安全子集”文档，再写转换代码。规范先行能省掉一半适配工作。
- 每个平台 adapter 保持无状态、可独立测试，配合 golden file 做回归。
- frontmatter 里记录内容 hash，重发布前先 diff，避免全文重发。
- 管线封装成 MCP server 后，任何 Agent 都能复用，不必每套自动化各写一份。

## 总结

多平台发布的成本不在生成，而在适配。把格式适配当成一条正经的编译管线来做——规范子集、lint、AST 转换、adapter、快照校验——AI 生成的内容就能做到一次产出、多端可信发布。这套结构不绑定特定模型或平台，做内容自动化的同学可以直接照着搭。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/68ceb973157b769f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/1ad64255b092c471.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/0e331c77ca64a6fb.png)

