---
title: 一条 Markdown 管线搞定多平台发布：Agent 产出的格式适配实践
feedId: 37256
source: 综合讨论
publishedAt: 2026-09-12
---

## 背景

我们用 OpenClaw 做内容自动化：Agent 查资料、写初稿、走审核，最后人工确认发布。链路看着很顺，卡住我们的反而是最后一公里——同一份 Markdown，发到公众号、博客、GitHub README、RSS，呈现效果各不相同。于是我们把它抽象成一条格式适配管线，本文记录做法。

## 问题

AI 生成的 Markdown 默认不是"干净"的：

- 混入原生 HTML：`<br>`、`<span style=...>`，渲染器不支持就直接裸露
- 标题层级跳跃：`#` 之后直接 `###`
- 代码块语言标签不稳定，同一语言有 `js` / `javascript` 两种写法
- 图片用本地路径或空 alt，跨平台后断链
- 数学公式、脚注、任务列表的支持矩阵各家不一

平台侧同样麻烦：公众号编辑器会剥掉大部分属性和类名，静态站认 front-matter，知乎对表格渲染保守。问题本质是：**AI 输出的是"大概能渲染"的 Markdown，而发布需要的是"确定能渲染"的目标格式。**

## 做法

管线拆成三层：

1. **规范化层**：定义内部规范格式（CommonMark + GFM 子集，明确禁用裸 HTML、标题最深三级、图片必须外链且 alt 非空）。这份规范写进 Agent 的 system prompt，让生成端就尽量合规。
2. **校验层**：基于 remark 生态写 lint 规则（`remark-lint` + 自定义 rule），Agent 产出后立即检查。不合规打回重写而不是人肉修补——反馈闭环放在自动化里。
3. **渲染层**：一份 canonical Markdown，按平台走各自的 remark / rehype 插件链。公众号走 rehype 转内联样式，博客走模板注入 front-matter，README 直接透传。

关键是 canonical 格式作为唯一事实来源，平台差异全部收敛在渲染层配置里。

## 踩坑点

- **AI 爱加行尾双空格和 `<br>`**：单独一条 rule 抓不全，建议 lint + 渲染前 sanitize 双保险
- **mermaid 和公式**：别指望全平台兼容，我们统一在渲染层转成图片，发布端不再依赖前端库
- **lint 打回重写容易死循环**：给 Agent 的报错必须可执行（比如"移除第 3 行的 HTML 标签"），重写上限设 2 次，超过转人工
- **图床防盗链**：外链图片在公众号后台上传后 URL 会变，这一步目前仍是脚本辅助 + 人工确认

## 可复用建议

- 光写规范文档没用，规范要落成两份制品：Agent 的 prompt 片段 + lint 规则，二者从同一份配置生成，避免两边漂移
- 每个平台渲染器配 golden files 做快照测试，升级依赖时 diff 一眼可见
- Agent 接了 MCP 的话，可以把 lint 和各平台渲染器包成 tool，让 Agent 在发布前自检一轮

## 总结

格式适配不是"找个转换器"的事，而是规范、校验、渲染三层职责的拆分。经验是：**在生成端约束得越早，渲染端需要补的洞越少**。管线跑通后，多平台发布的人工介入只剩最后的预览确认；升级依赖有快照兜底，改动收敛在渲染层配置里，整体链路稳定可控。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/e5f99dc6c85547c4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/f1f85fe492ff67a0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/84d91202d1eb514c.png)

