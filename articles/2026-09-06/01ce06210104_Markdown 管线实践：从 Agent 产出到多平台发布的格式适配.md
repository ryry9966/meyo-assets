---
title: Markdown 管线实践：从 Agent 产出到多平台发布的格式适配
feedId: 36350
source: 综合讨论
publishedAt: 2026-09-06
---

## 背景

我们社区里不少工作流已经变成：Agent 写初稿 → 人改 → 发到公众号、知乎、掘金、GitHub、静态站。前两步自动化程度很高，最后一步往往退回手工——复制粘贴、调样式、传图，每发一次丢半小时。

## 问题

根因不是“排版麻烦”，而是格式不兼容：

- 公众号不认标准 Markdown，只接受内联样式的 HTML；
- 知乎、掘金支持 GFM 但细节各异，脚注、任务列表、Mermaid 基本没戏；
- 静态站（Hugo/VitePress）几乎全支持，还多了 front matter；
- AI 产出的 Markdown 语法上合法，但不可移植：裸 HTML、相对路径图片、超宽表格、随意嵌套列表。

如果让 Agent 针对每个平台各生成一版，等于放弃单一事实来源，改稿时必然漂移。

## 做法：四段式管线

**1. 生成层：约束产出。** 写一个 skill / 系统提示，要求 Agent 只输出 GFM 子集：禁裸 HTML、图片用占位符、代码块必须标语言、表格不超过 4 列。宁可在源头少写，不在下游兜底。

**2. 规范化层：AST 检查。** 用 remark/mdast 做 lint，而不是正则。校验规则收敛成清单：非法节点直接报错或降级转换。这一步输出“规范 Markdown”，是所有下游的唯一输入。

**3. 适配层：按目标渲染。** 每个平台一个纯函数 transformer：

- 公众号：md → HTML，全部内联样式，代码高亮也内联；
- 静态站：保留 front matter，启用脚注扩展；
- 知乎/掘金：宽表格降级为列表，Mermaid 预渲染成 SVG 再上传。

**4. 资产与发布层。** 图片统一走图床：适配层把本地路径改写为外链，上传失败就中止，而不是发出死链。发布前先跑 dry-run 渲染 HTML 预览，人过目后推进草稿箱，不直发。

整条管线封装成 MCP 工具，Agent 只调用 `publish --target=xxx --dry-run` 这类接口，不接触内部实现。

## 踩坑点

- 公众号从剪贴板粘贴会丢样式，class 全部无效，必须内联 style；
- 列表项内嵌粗体加多级缩进，部分平台渲染直接断层级，lint 里限制嵌套不超过两层；
- 图床改写要在渲染前做，渲染后再替换容易踩转义字符；
- Mermaid 各平台都不支持，别指望客户端渲染，提前出图；
- AI 爱生成“看起来对”的宽表格，移动端必炸，4 列是红线。

## 可复用建议

- 单一母本 + 每平台一个 transformer，改稿只改母本；
- lint 规则即清单，Agent 发布前自检一遍，规则演进只改清单；
- 渲染产物按内容 hash 缓存，重复发布不重复计算；
- dry-run 默认开启，直发必须是显式参数。

## 总结

格式适配的本质，是把“平台差异”从人脑搬进管线。生成层收敛格式、AST 层把关、适配层各吃各的、资产层统一走图床——四段各司其职之后，从 Agent 写完到发出去，人工介入只剩“看一眼预览”。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/78d68a9ebbb8201f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/00e1d3da89c98b6b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/3a6584ba6b28b00f.png)

