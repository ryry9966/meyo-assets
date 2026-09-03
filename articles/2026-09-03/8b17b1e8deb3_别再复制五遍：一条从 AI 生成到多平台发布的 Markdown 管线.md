---
title: 别再复制五遍：一条从 AI 生成到多平台发布的 Markdown 管线
feedId: 35968
source: 综合讨论
publishedAt: 2026-09-03
---

## 背景

用 Agent 生成技术帖之后，真正花时间的往往不是写，而是发：公众号要内联样式、外链图片会裂；掘金、知乎的编辑器各有脾气；静态博客又要另一套 frontmatter。同一篇 Markdown，复制五遍、手调五遍，很快就会失控。

## 问题

直接拿 AI 输出去发，通常踩三类坑：

1. **格式不稳定**：模型有时夹带 HTML 片段，有时标题从 h3 开始，mermaid 块、脚注、任务列表混着来；
2. **平台能力差异**：公众号不支持锚点和脚注，代码高亮的 class 会被剥掉；宽表格在手机端直接溢出；
3. **图片防盗链**：外部图床的图在公众号里大概率显示不出来。

根因是把 Markdown 当纯文本做正则替换——文本层补丁越打越多，管线迟早烂掉。

## 做法

我把它拆成三层，全部脚本化（unified/remark 生态 + 一份平台配置）：

**1. 归一化层（normalize）**

不管谁来生成，先过一遍 mdast：

- 解析成 AST，降级或丢弃不支持的节点：HTML 节点转纯文本，mermaid 预渲染成图片；
- 标题层级归一：全文唯一 h1，其余顺延；
- 代码块补 language 标记，检查转义字符；
- 图片链接统一重写到自有对象存储。

**2. 平台档案层（profile）**

每个目标平台一份 YAML 声明能力矩阵：

```yaml
wechat:
  footnotes: false
  table: narrow        # 宽表格降级为列表
  code: inline-style   # 高亮构建期内联
  images: upload       # 走素材库 API
blog:
  footnotes: true
  table: native
```

渲染器按档案走：公众号输出"高亮内联 + 样式内联"的 HTML，博客基本直通，掘金裁掉不支持的语法即可。

**3. 发布层（publish）**

统一入口 `publish --to wechat,blog --dry-run`：dry-run 输出每个平台的渲染 diff，人工确认后才真正推送。Agent 侧把它包成一个 MCP tool，发布动作始终由人触发。

## 踩坑点

- **别指望提示词解决一切**。给模型定输出契约（标题层级、禁 HTML、图片占位符）能消掉 80% 的问题，剩下 20% 必须靠 AST 校验兜底；
- **公众号代码高亮必须构建期内联**，运行时靠 class 是无效的——style 属性会保留，class 会被剥；
- **mermaid 一定要预渲染**，各平台几乎都不支持，渲染成 SVG 再转 PNG 最稳；
- **表格是重灾区**，移动端优先的平台宁可降级成列表；
- **发布要幂等**：以 slug 为键记录已发布版本，避免 Agent 重试导致重复发文。

## 可复用建议

- 单一事实源 + N 个渲染器，绝不维护 N 份拷贝；
- 适配规则全部进配置，代码里只留通用逻辑；
- 每步产物落盘（`normalized.md`、`rendered/*.html`），出错可以从中间产物重放，不必重跑全管线；
- 接入新平台 = 新增一份 profile，不动管线代码。

## 总结

这条管线的核心不是"自动发布"这个动作，而是把格式适配从人肉经验变成"声明式配置 + AST 变换"。跑通之后，AI 生成 → 归一化 → 渲染 → dry-run → 发布全程脚本化，单篇多平台分发的边际成本基本归零。建议先打通两个平台再按需加档案，比一上来铺五个目标务实得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/5c3df0f90b52202f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/19562ea37ca41e6e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/fd651624c7e1b6ed.png)

