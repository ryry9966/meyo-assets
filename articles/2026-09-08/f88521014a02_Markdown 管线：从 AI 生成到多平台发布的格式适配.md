---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 36551
source: 综合讨论
publishedAt: 2026-09-08
---

## 背景

用 Agent 写技术帖效率不低，但 AI 产出的是“一份” Markdown，发布目标却是“多份”：公众号、知乎、掘金、静态站各有格式约束。手动逐平台调格式，偶尔一次还行，频率上来就是纯体力活。

## 问题

各平台对 Markdown 的支持差异很大：

- 公众号会清洗 HTML，外链图片、部分标签直接失效，代码块样式必须内联；
- 静态站要求 frontmatter（title/date/tags），AI 输出不一定符合 schema；
- 脚注、任务列表、Mermaid、公式的兼容性参差不齐；
- 图片是最常见的断点：本地相对路径、防盗链、上传后 URL 回写。

核心矛盾在于：AI 产出的是内容源，平台要的是投递格式。两者不该耦合在一份文件里。

## 做法

我把整条链路拆成四段，源文件只维护一份：

1. **生成层**：给 Agent 一个固定 prompt 契约——输出纯 Markdown、正文从 H2 开始、frontmatter 字段固定、图片一律用占位符。契约越死板，下游越省事。
2. **规范层**：markdownlint / remark-lint 过一遍，修标题层级、清理空行、校验 frontmatter schema。这一步失败就打回重生成，不要人肉修。
3. **转换层**：用 remark/rehype 按目标平台各建一条管线。公众号走 rehype-stringify 加内联样式插件，静态站走 remark-stringify 保留 Markdown。图片在这里统一下载、上传对象存储、回写 URL。
4. **校验层**：产物落盘后做快照 diff，对标题数、图片数、代码块数做断言，异常即中止发布。

我把它封装成一个 OpenClaw 技能，Agent 写完正文后调用一次命令，产物按平台分目录输出，人工只做最后过目。

## 踩坑点

- remark-stringify 重新序列化时会改写强调符号（`*` 换 `_`）并转义特殊字符，配置 `bullets` / `escape` 选项，否则“幂等”是假的；
- frontmatter 中文含冒号必须加引号，YAML 解析失败是最常见的静默错误；
- 公众号对表格支持很差，宽表格在移动端必炸，转图片或改成列表更稳；
- 代码块语言标记会被部分平台吞掉，高亮得靠转换层自己内联；
- 图片 URL 回写要测 alt 文本含括号的转义场景。

## 可复用建议

- 平台差异全部收敛到转换层，生成层永远只产出一种规范格式；
- 每条管线配 fixture 测试：输入固定样例，断言输出关键结构；
- 管线必须幂等——同一输入跑两次结果一致，否则没法接自动化；
- 图片按内容寻址当资产管理，上传与回写独立成步，别和文本转换耦合。

## 总结

多平台发布的成本不在写作，在格式。把 AI 当内容源、转换规则当代码维护、校验前置，发布就从手工活变成构建产物。管线不长，但每层都要契约明确、可断言，这才是它能在自动化里稳定跑起来的前提。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/32d28a68fcd6f24d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/fe1d23f61b4dc62f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/a360ededee9e71e1.png)

