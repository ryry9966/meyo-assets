---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 35969
source: 综合讨论
publishedAt: 2026-09-03
---

## 背景

用 Agent 写技术内容已经不稀奇：模型出初稿，人工校对，然后发布。真正麻烦的是发布这一步——同一篇 Markdown，博客要 frontmatter，公众号要转成带内联样式的富文本，掘金/知乎对表格和代码块的支持各不相同，GitHub 上还得保证相对链接不失效。

## 问题

两个来源的不确定性叠加：

1. **AI 输出不稳定**。同样是"写一篇技术帖"，这次从 h1 开始，下次从 h3 开始；代码块语言标签有时给有时不给；偶尔混入内联 HTML 或滥用加粗。靠 prompt 约束只能缓解不能根治——模型是非确定性的，管线必须是确定的。
2. **平台方言差异**。公众号会剥掉几乎所有的 class 和大部分标签；部分平台不支持 GFM 表格；图片路径（本地相对路径 vs 托管 URL）要求也不同。

结论：不要指望模型"一次生成适配所有平台的内容"，而是让它产出一份规范化 Markdown，格式适配全部交给确定性代码。

## 做法

管线拆四段，每段独立可测：

**1. Normalize（规范化）**
- heading 层级归一：正文强制从 `##` 开始，不允许跳级
- 代码块补齐语言标签，缺省就标 `text`，不留空
- 全角/半角标点、智能引号统一
- 用 remark 的 AST 处理，不要正则替换 Markdown——正则处理嵌套结构必然出错

**2. Transform（元数据抽取）**
- 抽出 title/description/tags 写入 frontmatter，删掉正文里重复的标题行
- 图片改写为平台要求的绝对 URL

**3. Render（渲染适配）**
- 每个平台一个 adapter，纯函数：`md → 平台格式`。公众号走 MD → HTML → 内联样式；不支持表格的做 table → 列表降级
- adapter 之间不共享状态，方便单测

**4. Verify（校验）**
- 渲染后检查外链可访问、图片非 404、长度不超平台限制
- 对渲染产物做快照测试，adapter 改动 diff 一眼可见

在 OpenClaw 里可以把这四段封装成一个 skill 或 MCP tool：Agent 只负责产出初始 Markdown 和调用管线，格式细节完全不进 prompt。

## 踩坑点

- **别在 prompt 里解决格式问题**。试过让模型"输出必须从 h2 开始"，成功率九成，剩下那一成会让下游渲染崩掉。prompt 管内容，代码管格式，边界要清晰。
- **下划线转义**。`snake_case_name` 在部分渲染器会被吃掉，normalize 阶段统一处理。
- **frontmatter 泄漏**。某个 adapter 忘了剥，发布出去就是 `---` 开头的尴尬内容。在 render 前统一剥离，别指望每个 adapter 都记得。
- **图片是重灾区**。相对路径在公众号直接失效；先上传拿 URL 再替换，上传失败要中断而不是留死链。
- **快照不是银弹**。HTML 快照对空白字符敏感，建议快照 normalize 后的 AST 哈希或关键节点。

## 可复用建议

- 一份 canonical Markdown 作为 single source of truth，所有平台产物都是派生物，可随时重新生成
- normalize 规则写成 remark 插件，跨项目复用
- 每条规则配最小 fixture（输入/期望输出），新增规则时跑全量 fixture 防回归
- adapter 保持无状态、幂等：跑两遍结果必须一致

## 总结

AI 负责生成内容，管线负责格式确定性，边界划清之后，多平台发布就从"每次手工调格式"变成"跑一遍脚本"。这套分层不限于写作场景：任何"LLM 输出 → 结构化下游消费"的链路，都适用同样的思路。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/7d60196fe092886a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/dfe6f80e3c1c79c2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/973b75929f5a07d6.png)

