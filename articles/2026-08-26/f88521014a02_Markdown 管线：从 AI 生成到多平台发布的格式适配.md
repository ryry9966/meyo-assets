---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 34744
source: 综合讨论
publishedAt: 2026-08-26
---

## 背景

在 OpenClaw/Agent 工作流里，让模型生成 Markdown 已经成为很常见的默认路径：周报、发布稿、产品说明、文档草稿都可以由 Agent 产出。但内容生成后直接复制到公众号、Notion、GitHub、飞书或 WordPress，经常出现格式错乱。这些问题不一定来自模型“写得差”，而是 Markdown 方言不一致、元数据不完整、图片链接失效和平台能力差异叠加的结果。

## 问题

1. **AI 默认输出不是稳定子集**：标题层级可能从 H1 开始，正文里混入前导 H1；代码块嵌套、表格列数不齐、frontmatter 写法随意。
2. **不同平台对 Markdown 扩展的支持差异很大**：GitHub 对 GFM 友好，公众号和知乎对 Markdown 基本不友好，Notion 导入会忽略部分 HTML，飞书需要块结构。
3. **自动化链路缺少校验**：坏格式会顺着管线传播，等发布后再修，成本已经翻倍。

## 做法 / 步骤

### 1. 定义中间规范

只允许 CommonMark 加少量 GFM 扩展：表格、任务列表、带语言标注的 fenced code block、YAML frontmatter。禁用原始 HTML、脚注、emoji 短代码、base64 图片和不必要的嵌套引用。frontmatter 固定字段：

```yaml
---
title: "管线实践"
slug: markdown-pipeline
summary: "从生成到发布的格式适配"
tags: [automation, markdown]
platforms: [github, wechat, notion]
cover: https://cdn.example.com/cover.png
---
```

### 2. 把规范放进 Agent 上下文

提供一份 `md-spec.md` 片段，要求模型在输出前自查。例如：正文从 H2 开始，H1 只保留给文章来源；代码块语言不能为空；表格分隔行必须写满 `---`；不要使用脚注。

### 3. 增加校验层

用 markdownlint 和自定义脚本检查 frontmatter 类型、标题顺序、表格列数、图片 URL 是否 http(s)、代码块是否闭合。校验结果用 JSON 返回，错误时回传给 Agent 修复，最多两轮。

### 4. 中间格式到平台 adapter

源 Markdown 保留在 `source/`，每个平台一个转换器：

```
source/2025-01-01-post.md
generated/wechat/2025-01-01-post.html
generated/notion/2025-01-01-post.json
published/2025-01-01-post.json
```

GitHub 直接复制 md；公众号/知乎先用 pandoc 或自研脚本转 HTML，再清洗标签；Notion 和飞书用 API 分块写入。不要在一个转换器里堆满所有平台逻辑。

### 5. 图片处理

AI 输出中可能出现相对路径、本地路径或外链。发布前统一下载、压缩、上传到对象存储或平台素材库，再替换为最终 URL。外链要做可达性检查。

### 6. 发布状态管理

每次发布记录平台、内容哈希、时间和错误信息到 `published/`，便于幂等和回溯。

## 踩坑点

- **代码块里出现 ```** 会导致解析中断。提示模型使用四个反引号包裹含三反引号的代码。
- **frontmatter 的 tags 有时是字符串而不是数组**，校验器要显式转换或报错。
- **公众号会清洗样式和部分 HTML**，转出的 HTML 要保守，不要依赖 class 和外部 CSS。
- **Notion/飞书 API 有块数量限制**，长篇文档要分批提交；失败后记录 offset，不能整体重试。
- **AI 生成的链接可能是幻觉链接**，发布前至少检查域名是否可解析。
- **不要在转换层修业务内容**，内容问题应回到生成层处理，否则多平台会各自长出不统一补丁。

## 可复用建议

- 把 `md-spec`、校验器、adapter 拆成独立模块，避免绑定某个具体 Agent。
- 用 frontmatter 作为唯一元数据入口，平台标题、摘要、标签都从这里取。
- 采用“生成-校验-修复”循环，最多重试两次，第三次转人工。
- 保留原始 AI 输出和转换产物分离，源文件不要被平台 adapter 修改。
- 为每个平台维护能力矩阵：哪些 Markdown 特性支持、哪些需要降级，如表格可转图片或列表。

## 总结

Markdown 管线真正要解决的问题不是“怎么多平台发布”，而是先让 AI 输出进入一个可控的中间规范。校验层和 adapter 分离后，格式问题可以被提前拦截，平台适配也能独立演进。工程上越早收敛格式子集，后续自动化越不容易失控。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/a1e75834bcc92145.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/6ecf01ac034657cc.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/24813cef527e4020.png)

