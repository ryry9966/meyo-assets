---
title: 从 AI 输出到多平台发布的 Markdown 管线实战
feedId: 31100
source: 综合讨论
publishedAt: 2026-07-31
---

## 为什么需要一条 Markdown 管线

OpenClaw 或 Agent 输出 Markdown 已经非常普遍，但当你试图把同一份内容发布到微信公众号、知乎、掘金或 Notion 时，噩梦就开始了——代码块丢失高亮、表格在移动端炸裂、图片链接失效、列表嵌套被吞。手工修复格式的成本远高于写作本身。

这就是需要一条“Markdown 管线”的原因：把 AI 生成的原始 Markdown，通过可配置、可复用的转换流程，输出为各平台干净的富文本或特定格式。管线不是“一键美化”，而是工程化的格式适配层。

## 问题拆解：平台差异到底在哪里

以一次日常发布为例，我们遇到的具体差异包括：

- **微信公众号**：编辑器是伪富文本，不支持直接粘贴 Markdown，代码块只会变成纯文本；有序列表里无法嵌套代码块；表格宽度需手动调整。
- **知乎**：支持部分 Markdown，但数学公式需要 `$$...$$` 且渲染器与 Typora 不同；代码块默认无行号；外链图片偶尔被屏蔽。
- **掘金**：Markdown 支持较好，但图片必须上传到其图床，无法直接使用外部链接；某些 HTML 标签会被清洗。
- **Notion**：导入 Markdown 会丢失元数据，表格支持差，代码高亮依赖 Notion 自身的语言检测。

AI 生成的 Markdown 常常包含：多级标题、代码块（带语言标注）、表格、内联 HTML、emoji、脚注等。这些“标准”元素在不同平台上的表现大相径庭。

## 管线设计：从 AST 到多目标输出

核心思路是：解析 Markdown 为标准 AST，然后为每个目标平台编写一套遍历规则，将节点转换为该平台接受的格式。技术栈如下：

- 解析器：unified + remark-parse（或 markdown-it）
- AST 操作：unist-util-visit 遍历节点
- 输出：针对公众号输出 HTML 片段（需符合其白名单），知乎输出调整后的 Markdown，掘金输出 Markdown + 本地图片替换，Notion 输出 Notion API 可识别的块格式。
- 桥梁：封装为 CLI 或 MCP server，让 Agent 可以通过工具调用，直接输出适配后的文本。

管线的示意流程：

```text
AI Markdown → 解析为 AST → 平台规则预处理器 → 节点转换 → 格式化输出 → 平台粘贴/API 发布
```

## 实现步骤（以 Node.js 为例）

1. **搭建项目**  
   `mkdir md-pipeline && cd md-pipeline && npm init -y`  
   安装依赖：`npm install unified remark-parse remark-stringify unist-util-visit`

2. **解析原始 Markdown**  
   ```js
   import {unified} from 'unified'
   import remarkParse from 'remark-parse'

   const tree = unified().use(remarkParse).parse(mdText)
   ```

3. **编写平台转换插件**  
   以微信公众号为例，需要处理代码块转为带 `white-space: pre-wrap` 的 `<pre>` 标签，并去掉语言标记（公众号不支持）。同时将图片 `src` 替换为已上传到图床的 HTTPS 链接。

   ```js
   import {visit} from 'unist-util-visit'

   function wechatPlugin(options) {
     return (tree) => {
       visit(tree, 'code', (node) => {
         node.type = 'html'
         node.value = `<pre style="...">${escapeHtml(node.value)}</pre>`
       })
       visit(tree, 'image', (node) => {
         node.url = uploadToCDN(node.url) // 自定义图床上传
       })
     }
   }
   ```

4. **输出目标格式**  
   对于公众号，最终需生成一个纯 HTML 字符串。可以使用 `remark-rehype` 再 `rehype-stringify`，但为了精确控制，常直接拼接。

5. **集成到 Agent**  
   将管线封装成一个 HTTP 服务或 MCP 工具，接收 `{ "md": "...", "platform": "zhihu" }` 返回处理后的内容。在 OpenClaw 中，你可以将这个工具注册为 MCP server 函数，让 Agent 在发布阶段自动调用。

## 踩坑记录

- **公众号列表嵌套混乱**：其编辑器会把 `<li>` 内部的块级元素（如 `<pre>`）强行拆开。解决办法是在代码块外用 `<section>` 包裹，并添加内联样式 `display: list-item`，但依然无法保证在移动端完全一致。最后决定“遇到列表中的代码块，升级为引用块+代码”的降级策略。
- **知乎图片防盗链**：外部图片可能返回 403。管线中统一使用图床，并且确保返回 `image/jpeg` 等 Content-Type。再增加 `referrerpolicy="no-referrer"` 属性有时能救急。
- **代码高亮丢失**：多数平台不保留 Markdown 的语法高亮。解决方案是在管线中集成 Shiki 或 highlight.js，将代码段渲染为带内联样式的 HTML 后再输出。代价是体积膨胀，但这是唯一让高亮幸存的方式。
- **表格列宽失控**：公众号表格列宽需设置百分比，且总宽不能超过 100%。管线里需要解析表格列数，自动计算列宽并添加 `style="width:XX%"`。

## 可复用建议

1. **规则配置化**：不要为一两个平台的差异写死逻辑。用 `platformRules.yaml` 或 JSON 配置描述节点转换策略，管线根据平台名加载规则。
2. **利用 unified 生态**：大量转换插件已有现成的，如 `remark-gfm` 处理表格和任务列表，`remark-math` 处理公式。优先组合，少造轮子。
3. **发布为 MCP 工具**：将管线做成可被 Agent 调用的标准 MCP service，输入原始 md 和目标平台，返回适配后的文本。这样任何支持 MCP 的客户端（如 OpenClaw 的 Agent 插件）都能直接使用，不必在每个项目里重复搭建。
4. **保留原始 AST 缓存**：如果后续要增加新平台，不需要重新从 Markdown 解析，直接从缓存的 AST 开始转换，提升效率。
5. **监控平台规则变化**：这些平台的编辑器特性并非固定，微信后台偶尔会调整 HTML 白名单。建议定期回归测试关键场景，保持管线更新。

## 总结

从 AI 生成 Markdown 到多平台发布，格式适配不是一个简单的一键转换问题，而是一系列琐碎规则的集合。把处理逻辑抽象为一条可配置的 Markdown 管线，用工程化的方式复用这些规则，既能避免重复劳动，又能让发布流程真正自动化。对 OpenClaw 用户来说，将这条管线封装为 MCP 工具，能让你的 Agent 从“生成文本”升级到“生成可直接发布的文本”，效率提升不止一个档次。

---

