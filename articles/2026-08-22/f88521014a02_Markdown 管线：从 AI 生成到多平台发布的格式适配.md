---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 34109
source: 综合讨论
publishedAt: 2026-08-22
---

## 背景

在 OpenClaw / Agent / MCP 自动化实践里，让模型输出 Markdown 很容易，但真正发布到公众号、知乎、掘金、Notion、飞书、WordPress 时，经常出现标题层级错乱、代码块语言丢失、表格断裂、图片外链失效、frontmatter 泄漏到正文。原因不是模型写得不够好，而是 Markdown 只是中间书写格式，不同平台是差异很大的消费端。

如果把 AI 输出的 Markdown 直接当最终交付物，后面每个平台的兼容问题都会变成一次性补丁，脚本越写越乱。更稳妥的做法，是把 Markdown 当成 IR（中间表示），建立一条可校验、可转换、可按平台发布的管线。

## 问题

常见的坑集中在四类：

1. **AI 输出不规范**：代码围栏不标语言、标题从 H1 跳到 H3、列表缩进混用、正文混入 HTML。
2. **平台方言差异**：公众号基本不认 Markdown，需要内联样式 HTML；知乎、掘金支持 GFM 但细节不同；Notion、飞书需要 block 结构；WordPress 适合 REST/HTML 导入。
3. **图片不可控**：AI 给的外链可能临时失效，本地图片、base64 图片需要统一上传替换。
4. **元数据处理**：frontmatter 中的特殊字符、嵌套结构容易解析失败，或者被平台显示在正文里。

## 做法

### 1. 定义中间 Markdown 子集

不要试图支持所有 Markdown 扩展。只允许 CommonMark + GFM 子集：标题、段落、有序/无序列表、任务列表、带语言标注的 fenced code block、表格、链接、图片、粗斜体、删除线、引用。

脚注、Callout、内嵌 HTML、复杂 LaTeX 默认不作为稳定输入。如果模型生成了，先转写或降级，不要直接透传。

### 2. 用 frontmatter 管理元数据

用 YAML 保存 title、slug、tags、summary、cover、published_at、platforms 等。解析后从正文剥离，避免进入平台正文。

```yaml
---
title: "Markdown 管线实践"
slug: "markdown-pipeline-multiplatform"
tags: ["markdown", "mcp", "automation"]
platforms: ["wechat", "zhihu", "notion"]
---
```

### 3. 生成后做 preflight 校验

用 markdownlint-cli2 或 remark-lint 配规则，检查标题顺序、代码块语言、列表围栏。再写一个轻量脚本校验 frontmatter 合法性、图片 URL 可达性、表格列数一致性、代码语言白名单。

校验不通过直接打回给 Agent 重写，比事后修更省事。

### 4. 建立转换层

用 unified / remark + remark-gfm 解析 AST，按平台输出：

- 公众号：remark-html 转 HTML，再套内联样式模板。
- Notion / 飞书：把 AST 转为 block payload，通过 API 创建。
- WordPress：输出 HTML 或走 REST API。
- 知乎 / 掘金：优先使用平台 API，不行再生成干净的 Markdown/HTML 供粘贴。

不要用 Pandoc 一把梭。Pandoc 适合文档转换，但会吞掉你需要保留的 frontmatter、注释或平台属性，反而增加黑盒。

### 5. 图片统一处理

管线中先收集 image 节点，区分 local、base64、external URL。本地图片上传到 OSS 或平台图床，替换为稳定 URL；base64 先解码成临时文件；外部链接做可达性检查，不信任 AI 临时图床。

### 6. 通过 MCP / 插件发布

把适配后的 payload 交给 MCP 工具或平台插件发布。关键原则：**只发布转换产物，不回读平台 HTML 再反向转换**，避免状态漂移。

## 踩坑点

- 公众号代码块必须转成带白底/灰底的 `pre`，列表需要加内联样式，否则 iOS/Android 渲染不一致。
- 知乎、掘金对图片防盗链敏感，外链图片最好先上传到平台。
- Notion / 飞书的代码块 language 有固定语言名，`jsx` 可能非法，`javascript` 才能通过；表格每行列数必须完全一致。
- frontmatter 中包含 `:`、`#`、`---` 时，YAML 解析容易出错，需要合理引号包裹。
- AI 有时会输出整篇被 ```` ```markdown ```` 包裹的内容，解析前要先剥掉外层围栏。
- 自动链接如 `<http://...>` 在部分平台会原样显示，需要转成普通链接。

## 可复用建议

1. **把规范写进 system prompt**：给 Agent 一份“可发布 Markdown 规范”和示例，比事后清理更有效。
2. **用 golden fixtures 做回归测试**：保存同一个 `input.md` 和每个平台的期望输出，改 adapter 后跑 diff。
3. **adapter 独立成插件**：平台变更只改对应 adapter，不改管线核心。
4. **先固定两三个平台**：跑通完整发布闭环，再横向扩展，不要一次性支持所有平台。
5. **图片和元数据处理前置**：不要在发布后再交叉引用平台生成的 URL。

## 总结

Markdown 管线的核心不是“转换格式”，而是建立中间契约：把 AI 的不稳定输出关进校验层，用可测试的 adapter 输出到多平台。这样每个平台不再是一堆临时脚本，而是一条可持续维护的自动化路径。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/e9d4cf5dd25afc78.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/ed3781256100ddc2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-22/d3b82165d0475cbb.png)

