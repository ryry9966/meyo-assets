---
title: Markdown 管线实践：让 AI 产出一次生成、多平台可发
feedId: 36968
source: 综合讨论
publishedAt: 2026-09-11
---

## 背景

Agent 产出的内容天然是 Markdown，但在社区里做自动化发布的同学很快会撞上现实：同一篇稿子，公众号、知乎、掘金、静态站各吃一套格式。我们在 OpenClaw 的发布插件里做了一条格式适配管线，把「AI 生成」和「多平台发布」解耦，本文记录一下落地过程。

## 问题

- **平台方言不统一**：公众号编辑器会剥掉 class、不支持大部分 GFM 扩展；知乎对表格和代码块有自己的处理；静态站什么都吃，但要求 front matter 字段齐全。
- **AI 输出不确定**：模型随手就写「嵌套列表里套表格」「mermaid 图」「HTML 混排」，谁渲染谁翻车。
- **手动改格式不可持续**：逐平台手调一遍，改完的版本还会脱离版本管理，源头一更新全部重来。

## 做法

核心思路是把这事当成编译问题：定源语言、写转换器、加类型检查。

**1. 定 canonical 方言。** 约束 Agent 只输出 GFM + front matter，禁用 mermaid、脚注、内联 HTML，用自定义 markdownlint 规则卡死，违规即拒绝。

**2. 归一化。** 用 remark 解析成 AST，统一列表缩进（2 空格）、中英文标点、图片链接转绝对地址。这一步必须在最前面，后面所有适配器只面对干净 AST。

**3. 平台适配器。** 每个目标平台一个 remark/rehype 插件：

- 公众号走「AST → 内联样式 HTML」；
- 知乎降级表格宽度、转换代码块写法；
- 静态站只补齐 front matter 字段。

**4. 校验回路。** 把 lint + normalize 封装成一个 MCP tool，Agent 发布前必须调用；不通过就把错误信息回喂给模型自修，设重试上限。

**5. 溯源与幂等。** front matter 里记录来源指纹和目标平台，发布器靠它判断「已发不再发」：

```yaml
---
source_hash: sha256:a1b2...
targets: [wechat, zhihu, site]
---
```

## 踩坑点

- **front matter 别过 HTML 转换器**：pandoc round-trip 会把 YAML 搞碎，元数据的增删要在 AST 流程外单独处理。
- **公众号代码块**：必须内联样式，且 `white-space: pre-wrap` 要显式给，否则长行直接截断。
- **公式语法**：知乎吃 `$$...$$`，有的平台只认 `\(...\)`，在适配器层统一转，别指望模型每次记得。
- **图片先上图床再进管线**：相对路径一转 HTML 就是死链；部分图床有防盗链，预览正常、发布后挂。
- **转码陷阱**：中文全角标点、em dash 会在某些编码环节被静默替换，归一化要提前拦截。

## 可复用建议

- 只让 canonical 源文件进 git，平台版本是构建产物，不手改、不入库。
- 准备一份覆盖所有语法的「金样」文档，每个适配器跑 snapshot 测试，升级 unified 依赖前先跑一遍。
- 管线对未知语法要 fail loud——静默降级比直接报错可怕得多，你发现不了内容已经坏了。
- 校验工具做成 MCP tool 而不是文档规则，Agent 对「能调用的约束」的遵守率远高于「写在 prompt 里的约束」。

## 总结

格式适配本质是个编译问题：canonical Markdown 是源语言，AST 转换是编译器，lint + 校验回路是类型检查。让 Agent 只负责生成合规源文件，平台差异全部下沉到适配器层，发布自动化才跑得长。目前这套管线在我们侧稳定支撑日更发布，新增一个平台平均半天工作量——大头都花在踩那个平台独有的渲染坑上。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/38d8642d21b2a742.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/fa75d89881e20067.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/9c78ec22ad799154.png)

