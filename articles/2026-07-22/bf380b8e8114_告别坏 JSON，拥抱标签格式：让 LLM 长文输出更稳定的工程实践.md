---
title: 告别坏 JSON，拥抱标签格式：让 LLM 长文输出更稳定的工程实践
feedId: 30116
source: 综合讨论
publishedAt: 2026-07-22
---

## 为什么 JSON 在长文输出里经常“骨折”

在 OpenClaw、Agent 工具链或 MCP 插件的实践中，我们经常需要让 LLM 返回结构化数据，最常见的做法是要求模型输出 JSON。大多数情况下它工作得很好，直到你要求模型生成一篇包含多段自然语言、代码块、甚至 Markdown 表格的长文章。

这时候问题开始集中爆发：

- 文本中自带的未转义双引号把 JSON 字符串切断；
- 多行文本忘记加换行符 `\n`，导致解析器报错；
- 代码块中的花括号与大括号嵌套让 JSON 结构直接坍塌；
- 模型为了“补全” JSON，会在末尾偷偷加上多余的逗号或括号。

更让人头疼的是，哪怕重试多次，模型这一次输出“差不多”的 JSON，下一次又会在新的坑位翻车。对于自动化流程来说，这种不稳定性直接导致 pipeline 中断，必须依赖复杂的重试机制和手工写正则修复——事实上，你可能已经在一个叫“修 JSON”的函数上投入了比业务逻辑还多的时间。

核心问题在于：**JSON 的严格性本身就是长文非结构化内容的敌人**。自然语言天然包含各种会破坏 JSON 语法的字符，而 LLM 在生成长文时很难精确控制每一个 escape。

## 换个思路：用标签格式替代 JSON

我从一个失败的文档生成 Agent 项目里学到：放弃 JSON，改用类似 XML/HTML 的**自定义标签格式**，可以让解析鲁棒性直接提升一个量级。基本思路是让模型输出这样的结构：

```
<article>
<title>文章标题</title>
<content>
这里是多段自然语言，可以包含双引号、单引号、<尖括号>，甚至代码块。
```python
def hello():
    print("world")
```

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/53d6f60bbe28de36.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/8606818476a984f4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/ee1b4477106003b4.png)

