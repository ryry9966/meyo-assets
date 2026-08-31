---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 35545
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景

在 OpenClaw 的自动化工作流里，让 Agent 生成 Markdown 已经很常见：技术笔记、发布草稿、README 片段、周报。但 AI 原始输出通常不能直接发到不同平台——GitHub、公众号、飞书、Notion、知乎对 Markdown 的解析差异很大。一次生成、多端复用，最后往往变成“每个平台手动改一遍”。

把这件事自动化，不是做一个全能转换器，而是建一条可复用的格式适配管线：把不稳定的 AI 输出规整成受控的中间 Markdown，再按平台输出目标格式。

## 问题

先看 AI 生成 Markdown 的常见问题：

- 开场白和结尾：“好的，以下是你要的……” 以及 “希望对你有所帮助” 混在正文里。
- 标题层级跳跃：直接从标题一跳到标题三，或全文只有一个标题。
- 代码块没有语言标识，导致高亮失效。
- 表格分隔行缺失，渲染成普通段落。
- 图片链接是临时外链，过几天就失效。
- 混入前端 matter、内嵌 HTML、emoji 变体等，部分平台不识别。

再看平台差异：

- GitHub 支持 GFM 表格、任务列表，但不支持脚注、流程图、数学公式。
- 公众号不能直接解析 Markdown，需要转成带内联样式的 HTML，table 必须手动加边框和 padding。
- Notion/飞书通过 API 接收块结构，heading、code、table 都需要映射成各自的 block 格式，表格列宽受限。
- 知乎对部分 Markdown 语法支持有限，代码块语言标识不生效时高亮全无。

如果每次用正则替换或手工修，不仅成本高，而且不可复现，Agent 下次还是生成同样问题。

## 做法/步骤

管线的核心思路是：**约束 AI 输出 → 解析为 AST → 平台 Adapter 转换 → 校验**。

示意流程：

```text
raw_output.md
  ├─ preprocess: strip boilerplate, normalize headings, infer code lang
  ├─ parse: remark -> mdast
  ├─ transform: platform adapters (GitHub / WeChat / Notion / Feishu)
  └─ validate: lint rules + link check
```

**1. 定义安全 Markdown 子集**

作为中间格式，只保留：h2–h4、段落、有序/无序列表、代码块（必须带语言）、表格、图片、链接、引用、加粗/斜体。禁用脚注、流程图、数学公式、内嵌 HTML、复杂 callout。各平台 Adapter 决定如何降级处理超出子集的内容。

**2. 预处理**

用脚本或 Agent 工具链清洗原始输出：

- 截断开头“好的，以下是……”和结尾“希望对你有所帮助”。
- 修正标题从 h1 开始且不跳级。
- 用文件名或上下文推断代码块语言，补全缺失的 language。
- 统一空行数量，避免连续多个空行。
- 做 Unicode 归一化，清理零宽字符、全角符号混用。

推荐基于 remark 的 AST 遍历实现，而不是字符串正则。AST 能保证结构不被破坏。

**3. 平台 Adapter**

每个平台一个 Adapter，从 Markdown AST 映射到目标格式：

- **GitHub**：直接输出 Markdown，但做最小检查，确保表格分隔行存在、代码块语言合法。
- **公众号**：用 remark-rehype 转 HTML，给 table 注入内联样式（`border-collapse:collapse`，`td/th` 加边框和 padding），图片转存到 CDN 并替换链接，标题最多保留到 h3。
- **Notion/飞书**：将 mdast 节点映射为 block 对象（paragraph、heading_2、code、bulleted_list_item、table 等）。Notion 的 code block 必须提供 language 字段，飞书代码块不支持行号，需要降级处理。

**4. 校验**

用 lint 规则检查输出：是否存在不安全语法、图片是否有 alt、标题是否越级、表格列数是否一致、链接是否可访问。可以集成到 CI 或 OpenClaw 的 Agent 工具中，发布前自动跑一遍。

## 踩坑点

实际使用中最容易翻车的地方：

- **代码块语言丢失**：AI 经常只写 ` ``` ` 不写语言。预处理时没补全，转换后高亮全部失效。建议对未知代码块强制补 `text` 或按上下文猜测。
- **公众号表格样式丢失**：直接转 HTML 的 table 没有边框和单元格间距，发布后就是个无边框表格，用户会以为是转换器问题。Adapter 必须注入内联样式。
- **Notion/飞书 block 字段错误**：Notion 的 code block 需要 `language` 字段，如果传空字符串会导致创建失败；飞书表格对列宽有上限，超宽列需要截断或降级为图片。
- **图片链接时效**：AI 生成的占位图链接可能过几天就失效，发布前要校验可达性并转存。
- **混入 frontmatter**：AI 有时会输出 YAML frontmatter，GitHub 能忽略，但公众号、Notion 会渲染成正文。预处理必须剥离。
- **标题过深**：公众号、知乎对 h4 及以下标题折叠明显，Adapter 里最好统一限制到 h3，多余层级降级为加粗段落。

## 可复用建议

1. **用 unified/remark 生态做 AST 级转换**，不要用正则替换 Markdown。AST 能安全处理嵌套结构和转义。
2. **把每个平台配置写成 profile**：标题层级上限、是否支持表格、代码块语言列表、图片处理方式、特殊语法开关。Adapter 读取 profile 做适配，新增平台只加一份配置。
3. **用 MCP 封装管线的核心步骤**：暴露 `clean_markdown`、`convert_markdown`、`validate_markdown` 三个工具，让 OpenClaw 的 Agent 直接调用，而不是每次写临时脚本。
4. **保留原始输出和中间 Markdown**：调试时能对比清洗前后的差异，也方便回归测试。
5. **对 AI 输出加约束 prompt**：要求只输出安全子集、代码块必须带语言、不要开场白和结尾、标题层级从 h2 开始。从源头减少清洗成本。
6. **建立转换测试集**：收集典型 AI 输出样本，每个平台跑一遍对比渲染结果，尤其是表格、代码块、图片三类。

## 总结

Markdown 管线解决的不是“转换格式”而是“控制不稳定输出”。先约束 AI 生成、再走 AST 中间表示、然后平台 Adapter、最后校验，这套链路能把多平台发布从手工活变成 Agent 自动执行的步骤。在 OpenClaw 体系里，把它封装成 MCP 工具后，复用性和可维护性都会比一次性脚本高很多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/a6c87cae65eb9749.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/ad9c30e2a1aa9cf2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/51dc9813c72a2d24.png)

