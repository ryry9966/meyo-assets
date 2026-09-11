---
title: Markdown 管线：把 AI 生成的内容稳定发布到多个平台
feedId: 37118
source: 综合讨论
publishedAt: 2026-09-12
---

## 背景

在 OpenClaw 的 agent 工作流里，AI 的输出天然是 Markdown：技术博客、周报、教程、发布说明。问题出在发布端——公众号、知乎、掘金、自建博客对 Markdown 的支持程度完全不同。我们最初的流程是“生成完手动复制粘贴修一遍”，改到第三轮内容后就放弃了，于是把格式适配做成了管线。

## 问题拆开是三类

1. **语法子集不一致**：脚注 `[^1]`、任务列表、数学公式、多列表格，各平台支持参差不齐；
2. **样式策略不同**：公众号编辑器只认内联样式，会剥掉 class 和 script；掘金保留 GFM 但限制裸 HTML；
3. **资源引用**：本地图片路径失效、图床防盗链、平台二次压缩。

## 做法

管线分五步：

**1. 收敛源头。** 约束生成端只产出 Markdown 子集：h2–h4、带语言标记的围栏代码、GFM 表格不超过 4 列、不用脚注和裸 HTML。在 prompt / 输出 schema 里约束，比下游修补便宜一个量级。

**2. 规范化。** 用 remark/unified 做 lint + fix：统一空行、列表缩进、代码语言标记。这一步与平台无关，产出是唯一的 canonical Markdown。

**3. AST 层适配。** 在 mdast 上写平台 transform，而不是做字符串替换：

- 公众号：正文链接降级为文末参考列表；表格超 3 列转列表；代码块用 shiki 生成内联高亮 HTML；
- 知乎/掘金：保留 GFM，剥离自定义容器；
- 博客：全量保留，补 frontmatter。

**4. 资源解析。** 图片在 canonical 源里只用稳定 asset id，发布前按目标平台预上传并替换 URL。放在最后一步，避免适配过程反复改写引用。

**5. 接入自动化。** 封装成 MCP 工具 `publish(platform, doc_id)` 挂进 agent；lint 作为发布 gate，各平台渲染结果存 golden file，transform 改动跑快照对比。

## 踩坑点

- 公众号剥 class 和 `<script>`，样式必须内联，且粘贴后还有一层二次压缩，行高会变——别按本地渲染器预览盲调；
- `[^1]` 脚注语法绝大多数平台不支持，只能在 transform 里降级成文内标注；
- KaTeX 和 MathJax 渲染差异明显，部分移动端场景干脆不渲染，公式多的内容建议转图片或放弃该平台；
- 图片被平台重新压缩后清晰度不可控，关键截图提前按目标宽度出图；
- 最大的坑：让模型直接生成“平台特化版”。多份源文件必然漂移，改一处要同步 N 处，最后没人知道哪份是真的。

## 可复用建议

- 单一 canonical 源 + 单向适配函数，禁止适配结果回写源文件；
- 约束前移到生成端，下游只做机械转换，不做内容判断；
- lint 当 gate、golden file 当回归测试，比肉眼验收可靠；
- 资源与文档分离，URL 解析永远放管线最后。

## 总结

这条管线的工作量主要不在“写多少转换器”，而在把自由度收窄：源头约束子集，中间只有 AST transform，发布只是最后一层薄封装。收敛之后，新增一个平台通常只需要一个 transform 文件加一份 golden file；对 agent 来说，“写完”和“发出去”之间只剩一次可靠的工具调用。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/c4f6540c7181eeae.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/12c1150c34eda537.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/36e73a2b49da187c.png)

