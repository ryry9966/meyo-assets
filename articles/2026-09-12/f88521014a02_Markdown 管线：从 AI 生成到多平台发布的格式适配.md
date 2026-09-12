---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 37253
source: 综合讨论
publishedAt: 2026-09-12
---

## 背景

OpenClaw 场景下，Agent 产出天然是 Markdown：报告、周报、文档、教程。问题在于这些内容往往不止发一个地方——博客、公众号、知乎、GitHub README，各家编辑器对 Markdown 的支持差异很大。与其为每个平台手写一遍，不如搭一条管线：AI 生成一次，适配器分发多次。

## 问题

直接复制粘贴会遇到几类问题：

1. 公众号编辑器会剥掉 class 属性和大部分 HTML，代码高亮必须预先渲染成内联样式；
2. 知乎不支持脚注和部分任务列表语法；
3. AI 输出的 Markdown 本身不规范：混入内联 HTML、表格过宽、代码块语言标记缺失、图片引用指向本地路径；
4. 重复发布会产生重复草稿，缺少幂等机制。

## 做法

整体思路：**规范输入 → AST 中间层 → 平台适配器 → 校验发布**。

**第一步：定义规范 Markdown。** 把管线接受的语法写成白名单：GFM 基础语法 + 围栏代码块 + 图片占位符，明确禁止内联 HTML、脚注、深层嵌套引用。这份文档是整条管线的契约。

**第二步：约束生成端。** 在 Agent 的 system prompt 或 MCP 工具描述里注入同样的白名单，并要求图片一律用 `{{img:xxx}}` 占位符。生成端约束比后端清洗便宜得多。

**第三步：AST 中间层。** 用 unified/remark 生态把 Markdown 解析成 mdast，所有转换在 AST 上做纯函数变换，而不是正则替换字符串。比如把占位符节点替换成真实图床链接、把代码块节点交给高亮器渲染成带内联样式的 HTML。

**第四步：平台适配器。** 每个平台一个 adapter，核心差异在渲染策略：

- 博客/静态站：直接输出 Markdown + frontmatter；
- 公众号：mdast → HTML → 样式内联，图片先传素材库再替换 URL；
- 知乎：脚注降级为文末列表，去掉任务列表语法。

**第五步：校验与 dry-run。** 发布前跑两道检查：markdownlint（自定义规则集）+ 链接可达性。dry-run 模式输出各平台预览文件，人工确认后再真发。

## 踩坑点

- **不要用正则清洗 Markdown。** 嵌套结构的边界用正则几乎写不对，remark 几行就能做到且可测试。
- **公众号的代码高亮**必须离线渲染成内联 style，粘贴时才不丢色；行内代码和块级代码要分开处理。
- **表格在移动端必然溢出。** 管线里加一条 lint：表格列数超过四列就告警，或自动降级为列表。
- **幂等性靠稳定 ID。** 用 slug 而不是时间戳做文档标识，重复执行 publish 是覆盖而非新建。
- **remark 版本差异**会改变 AST 结构，务必锁版本，并用快照测试覆盖每个 adapter 的输出。

## 可复用建议

- 规范 Markdown 白名单同时给生成端和校验端引用，一处定义、两处生效；
- adapter 做成纯函数：输入 mdast + 配置，输出目标格式，方便单测和快照；
- 图片上传是唯一有副作用的步骤，独立成异步阶段，失败可重试；
- 对接 MCP 时，把 `publish(target, dryRun)` 暴露成工具，Agent 只管调用，格式细节全部收敛在管线里。

## 总结

这条管线的核心不是“多平台发布工具”，而是把格式适配从内容生成中剥离：AI 只负责产出符合契约的 Markdown，格式复杂度由 AST 中间层和适配器承担。管线上线后，新增平台只需写一个 adapter，内容端零改动。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/b71dc9164cd5fb00.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/7fa3597a68e7c91d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/093cd520bb2fbb22.png)

