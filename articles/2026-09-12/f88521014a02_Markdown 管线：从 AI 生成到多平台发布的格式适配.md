---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 37116
source: 综合讨论
publishedAt: 2026-09-12
---

## 背景

在 OpenClaw 的自动化场景里，让 Agent 产出内容只是第一步。真正麻烦的是后半程：同一段 Markdown，发公众号要转内联样式 HTML，发知乎会被吞表格，发静态站又要处理 frontmatter 和相对路径。我们团队把这条链路收敛成了标准管线，本文记录设计与踩坑。

## 问题

LLM 输出的 Markdown 看似规范，差异集中在三类：

1. **结构不稳定**：标题层级跳级、把整段输出包进代码围栏、frontmatter 字段漂移；
2. **语法超集**：脚注、任务列表、内联 HTML、KaTeX 公式——这些不是所有渲染端都认；
3. **中文排版**：CJK 与西文之间无空格、全半角标点混用，以及一个经典 bug：`**加粗**` 紧贴中文字符时，部分 CommonMark 实现因 flanking 规则不渲染。

下游平台再各自加一层限制。所以问题不是“有没有 Markdown”，而是“是哪一种 Markdown”。

## 做法

管线分四层，核心思想：**把 Markdown 当 AST 处理，而不是当字符串**。

1. **生成约束层**。在 Agent 的 system prompt 里固定输出协议：标题只允许二级以下、禁用内联 HTML、frontmatter 字段白名单、代码块语言限定在 whitelist。约束生成端永远比修复下游便宜。
2. **归一化层**。用 unified/remark 做 mdast 级 transform：统一标题层级、CJK 加空格（pangu 规则实现成 remark 插件）、为不支持的平台把脚注降级为文末引用列表、剥离危险标签。产出一份“内部规范 AST"。
3. **适配层**。每个平台一个 adapter，输入都是归一化后的 AST：公众号走 remark-rehype 再内联样式，静态站直接序列化回 MD，知乎做表格降级。adapter 之间互不感知。
4. **校验与分发层**。发布前跑 lint（markdownlint + 自定义规则）、死链检查、图片存在性检查，golden 文件做快照测试，通过后才交给发布插件或 MCP 工具（如 `publish_to(platform)`）。

## 踩坑点

- **正则修格式迟早出事**。嵌套代码块里的 "```" 会骗过文本级替换，改用 AST 后才稳定。
- **中英文加粗渲染不一致**。部分渲染器对 CJK flanking 处理不同，我们的方案是在归一化层统一转 `<strong>`（仅目标平台允许 HTML 时），adapter 层做开关。
- **图片相对路径**。Agent 生成的相对路径发布即死链，归一化层强制转绝对 URL 并校验可达。
- **frontmatter 里的冒号**。标题含 `：` 未加引号会解析失败，靠 lint 规则兜底。
- **公式双方言**。KaTeX 和知乎公式语法不同，别指望一处渲染处处可用，adapter 里各写一遍转换。

## 可复用建议

- 写一份“AI 输出 Profile”，贴进所有内容生成 Agent 的 prompt，全管线共用一份。
- adapter 保持纯函数：AST 进、格式出，不碰网络不碰文件，方便单测。
- 每个平台留 2–3 个 golden 样例，CI 里做 diff，平台编辑器改版第一时间能发现。
- 把适配能力包成 MCP tool，Agent 就能自己决策“这篇发哪、怎么发”。

## 总结

多平台发布的核心不是写更多转换代码，而是**收敛源头 + AST 中间表示 + 平台隔离**。把格式适配从“每个脚本各修各的”变成一条可测试的管线后，新增平台基本只是多写一个 adapter 文件的事。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/628bac4fdd93206d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/ca9f59749b9ba951.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/7e106e7a4ec7c976.png)

