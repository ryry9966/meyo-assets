---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 36097
source: 综合讨论
publishedAt: 2026-09-05
---

## 背景

在 Agent 工作流里，LLM 的输出天然是 Markdown，这一步很顺。不顺的是后面：同一篇内容往往要发到公众号、知乎、博客和 GitHub，各平台渲染器能力差异极大。让模型针对每个平台各生成一遍，内容很快漂移；更稳的做法是单源多出——一份 Markdown 源文件，一条管线负责适配。

## 问题

三个层面的痛点：

1. **方言不纯**。模型输出常见 ATX/Setext 标题混用、裸 HTML 片段、未闭合代码围栏、嵌套列表缩进漂移。
2. **渲染差异**。公众号会剥离 `<style>` 和脚本，代码高亮的类名全部失效，外链图片被防盗链；知乎会丢任务列表；静态站点最宽容。
3. **资源耦合**。图片路径、Mermaid、公式，都没法指望平台端 JS 帮你渲染。

## 做法

我们用 unified（remark/rehype）搭了四段式管线，用 markdown-it 插件化也能实现同样思路。

**1. 归一化**。统一 ATX 标题、GFM 表格，raw HTML 走白名单，未识别节点直接报错而不是静默吞掉。同时在 Agent 的 system prompt 里限定方言，管线侧用 markdownlint 加自定义规则兜底——两层约束，缺一不可。

**2. 元数据**。frontmatter 声明发布目标：

```yaml
---
title: ...
tags: [pipeline]
targets: [wechat, zhihu, blog]
---
```

管线按 `targets` 选择适配器，没列的平台不处理。

**3. 适配器**。每个适配器是纯函数：AST 进，平台载荷出。公众号适配器把 highlight 类名编译成内联色值的 span，图片先传素材库再替换 URL，Mermaid 和公式在服务端渲染成 PNG；知乎/掘金适配器只做节点裁剪；静态站透传加前置处理。

**4. 发布**。优先官方 API（如公众号草稿箱接口）；没有 API 的平台用剪贴板加浏览器自动化过渡。在 OpenClaw 里把整条链路封装成 MCP 工具：generate → normalize → adapt → preview → publish，preview 阶段强制人工确认，publish 不做自动重试。

## 踩坑点

- 不要用 HTML 转 Markdown 的反向转换来“适配”，往返必丢信息，所有变换都应在 AST 上做。
- 公众号客户端 CSS 支持不全：flex 在部分环境失效，表格要控列宽，长代码块会横向溢出。
- 外链图必须在发布前落到目标平台认可的存储，防盗链没有绕过捷径。
- 浏览器自动化的登录态会过期，重试要有上限并告警，否则排障时全靠猜。
- 代码块内容里出现 ``` 会截断围栏，AST 校验能兜住，纯文本正则兜不住。

## 可复用建议

- 先写一页“最小方言”规范加 lint 规则，比事后修复便宜得多。
- 适配器保持无副作用，用快照测试锁住各平台输出，平台改版时 diff 一目了然。
- 图片与内容分离：管线内部用稳定 ID 引用，发布时按平台解析成最终 URL。
- 预览优先：每个平台先出预览产物（HTML 或截图），人确认再发。

## 总结

单源多出的核心不是让模型记住平台规则，而是把平台差异收敛到适配器层。方言约束得越严格，管线越稳定，Agent 的产出也就越可预测。适配器写多了你会发现，真正难的不是 Markdown 语法，而是各平台对“合法内容”的定义——把这层定义固化成代码，就是这条管线最大的价值。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/d110ea8fd65bacb9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/b94bc328e7c85860.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/c49534361f634104.png)

