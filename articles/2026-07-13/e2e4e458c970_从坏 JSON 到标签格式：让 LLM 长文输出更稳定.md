---
title: 从坏 JSON 到标签格式：让 LLM 长文输出更稳定
feedId: 28889
source: 综合讨论
publishedAt: 2026-07-13
---

在 Agent 自动化流程中，让 LLM 输出结构化长文一直是常见的需求：生成报告、剧本、评测、配置等。很多人第一反应是让模型返回 JSON，但在生产环境跑几轮你就会发现——越是长输出、越复杂的嵌套 JSON，解析失败率越高。这里分享一种更务实的替代方案：用自定义标签格式替代 JSON，配合最小解析器，把稳定性从 “运气” 变成 “确定”。

## 背景：为什么长文 JSON 老是坏

JSON 作为数据交换格式没什么问题，但把它当作 LLM 的**原生输出格式**问题不少：

1. **括号匹配敏感**：长文本里一旦出现未转义的引号、花括号、方括号，整个结构直接坏掉。模型在生成一大段自然语言时，很难完全遵守转义规则。
2. **尾部截断**：长输出更容易被 max_tokens 截断，截在某个数组或对象中间，JSON 完全无法解析。
3. **修复成本高**：即使你接修复库（如 `json_repair`），对于结构性破坏（比如多出一个 key 却少了逗号），修复结果可能语义错乱。
4. **Prompt 设计矛盾**：为了强制 JSON，prompt 里需要反复强调格式，挤占了对内容质量的注意力，模型反而容易在内容里偷懒或出现幻觉。

特别是当你让模型输出上万字的剧本，里面包含对话、动作、场景描述，硬塞进 JSON，结果就是要么输出质量下降，要么一次解析成功率不足 70%。

## 标签格式的思路

放弃 JSON，改用类似 XML/HTML 的自定义标签来包裹不同的结构段。例如，让模型输出一篇文章大纲和正文：

```
<outline>
<item>背景介绍</item>
<item>问题分析</item>
<item>解决方案</item>
</outline>
<content>
<section title="背景介绍">
这里是自然语言正文……
</section>
<section title="问题分析">
这里又可以包含多段自然语言……
</section>

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-13/b0b3bbb2d73bd9cc.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-13/c12e3fba6b0fb47a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-13/ff5667884ea0f49d.png)

