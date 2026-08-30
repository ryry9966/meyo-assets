---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 35465
source: 综合讨论
publishedAt: 2026-08-31
---

在 OpenClaw/Agent 工作流里，让模型输出 Markdown 很容易，但把同一份 Markdown 发到微信公众号、知乎、博客、GitHub 等多个平台，格式往往面目全非。原因不是模型写得不好，而是各平台对 Markdown 的支持子集差异极大。本文记录一条可复现的适配管线：从源头约束 AI 输出，到中间转换，再到目标平台校验和发布。

## 背景

AI 生成的内容通常以 Markdown 形式返回，结构清晰，适合版本管理。但不同平台对 Markdown 的解析能力不同：有的只支持标题、加粗；有的支持代码块但不支持表格；有的对图片外链直接拦截；有的公式渲染要求严格。如果每次发布都靠人工复制粘贴调整，成本很高。

## 问题

直接发布会遇到几类典型问题：

- 代码块：微信不识别围栏语法，知乎高亮样式丢失。
- 表格：标准 Markdown 表格在部分平台被转义成纯文本或错位。
- 图片：外链会 404 或被防盗链，微信需要上传素材库。
- 公式：LaTeX 在多数平台不渲染，需要转 SVG 或图片。
- 链接与排版：外链不可点、列表嵌套缩进不一致、空行规则不同。

## 做法/步骤

1. **约束源 Markdown 子集**  
   定义允许的语法范围：ATX 标题、围栏代码块、有序/无序列表、链接、图片、表格、引用、加粗/斜体、删除线、水平线。禁止内嵌 HTML、自定义 class/style 和脚本。元数据用 frontmatter 承载（title、tags、summary、cover、platform_config）。

2. **解析为中间表示（IR）**  
   用 unified/remark 将源 Markdown 解析为 AST，不直接做字符串替换。这样后续转换只操作节点，不容易破坏嵌套结构。

3. **声明式平台规则**  
   为每个目标平台写一份 JSON 配置，例如：
   ```json
   {
     "platform": "wechat",
     "code_block": "pre_code_inline_style",
     "table": "html_table_or_image",
     "image": "upload_to_cdn",
     "formula": "render_svg",
     "external_link": "short_or_prompt"
   }
   ```
   新增平台只需要加配置，不改核心转换逻辑。

4. **转换 + 校验**  
   转换后跑 lint：检查未闭合标记、空链接、非法节点、图片地址可达性。对关键平台做一次渲染预览（可用 Playwright 截图或平台草稿箱检查）。

5. **发布钩子**  
   通过平台 API 或浏览器自动化发布。失败保留草稿，并写日志，便于回溯。

## 踩坑点

- **微信代码块**：不能直接用围栏，需要转成 `<pre><code>` 并内联样式；但微信会过滤 `style` 属性，行内空格容易丢失，建议用渲染图片或第三方格式化工具。
- **表格**：知乎、掘金对标准表格支持差，可转 HTML table 或截图；公众号则完全不可点表格内链接。
- **图片**：管线内需要先下载原图，再上传到各平台 CDN，最后替换 Markdown 中的图片链接，否则会出现防盗链或过期。
- **公式**：公众号不支持 LaTeX，需要预先渲染成 SVG 或 PNG；知乎支持部分 KaTeX，但语法不兼容，需要做转译。
- **列表嵌套**：有些平台要求列表前必须空行，否则子列表被压平；建议在转换时统一补充空行，而不是依赖原始 Markdown。

## 可复用建议

- 保持“一个源 + 多平台配置”，不要为每个平台维护手写副本。
- 把平台规则做成声明式 JSON，平台差异外置。
- 建立平台适配测试用例集：代码块、表格、图片、公式、嵌套列表、emoji、长文折叠，每次改规则后自动跑一遍。
- 在 OpenClaw 中用 MCP 封装转换器和发布器，Agent 生成 Markdown 后自动触发适配，减少人工介入。
- 保留原始 Markdown 和转换产物，便于问题定位与回滚。

## 总结

Markdown 管线的核心不是做一个“万能转换器”，而是通过受限的源、显式的平台规则和可校验的输出，把格式差异控制在小范围内。这样既保留了 AI 生成内容的效率，又能稳定地分发到多个平台。务实做法比追求全语法兼容更可靠。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/757c401be46349c3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/5f7fedeee4ce5884.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/7e13dcbf3c4170c2.png)

