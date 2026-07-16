---
title: 告别坏 JSON：用标签格式提升 LLM 长文输出的工程稳定性
feedId: 29279
source: 综合讨论
publishedAt: 2026-07-16
---

## 背景

在构建 Agent 插件或 MCP 自动化流程时，你大概率会让 LLM 输出包含大段文本的结构化结果。最直觉的做法是要求模型返回 JSON。例如：让 Agent 完成自动邮件撰写，输出 `{"subject": "...", "body": "..."}`；或者让工具返回一段长摘要与原文引用。问题很快就会出现——当生成的文本变长，JSON 就变得脆弱。

流式输出过程中，一个未转义的双引号、一段多行字符串里混入的反斜杠，或者模型偶尔“创作”出的换行，都会让 `json.loads()` 直接抛异常。更头疼的是，这些断裂经常发生在生产流水线的深夜，触发一连串重试与告警。在 OpenClaw 社区里，不少做自动化实践的伙伴都踩过这个坑。

## 坏 JSON 长什么样

举个真实的“小意外”：Agent 需要把用户输入的一段反馈包裹进 JSON 的 `detail` 字段返回。用户这句话里恰好有一处中文引号“”，或者英文双引号没做转义，输出就变成：

```json
{"summary": "一切正常", "detail": "用户说："太棒了"，点击确认"}
```

JSON 解析器立刻报错，因为字符串被非转义双引号提前终结。要是遇到更长的报告文本，情况只会更糟。你当然可以在 prompt 里强调“请转义双引号”，但实际测试中，长上下文的模型总会偶尔遗忘，尤其当它需要引用用户原文时。

## 换个思路：标签格式

工程上更稳健的做法，是放弃 JSON 作为长文本的容器，改用标签（类似 XML 的自定义标记）来结构化输出。比如：

```
<subject>周报 - 第42周</subject>
<body>
本周完成了 A/B 测试重构，用户反馈积极。注意后续需要处理"edge case"时的双引号问题。
</body>
```

你只需要在 prompt 里约定输出格式：

> 请将最终内容放在指定的 XML 标签中，不要使用 JSON。标题用 `<subject>` 和 `</subject>` 包裹，正文用 `<body>` 和 `</body>` 包裹。正文中的尖括号请替换为 `&lt;` 和 `&gt;`。

提取时，完全不需要 JSON 解析，只需要一次简单的正则：

```python
import re

def extract_tag(content: str, tag: str) -> str:
    pattern = rf"<{tag}>(.*?)</{tag}>"
    match = re.search(pattern, content, re.DOTALL)
    if match:
        return match.group(1).strip()
    return ""
```

这种做法回避了 JSON 里最折磨人的转义问题，因为标签边界清晰，正则非贪婪匹配直接拿到内部原始文本，不会因为双引号或换行而断裂。对于 MCP 工具链接口的响应，把结构化字段拆成不同标签也更便于下游解析，错误隔离性好得多。

如果你需要更丰富的结构，可以采用更具体的标签组合，例如：

```
<action>send_email</action>
<to>team@example.com</to>
<content>邮件正文，可以自由包含双引号"和换行。
多行内容完全没问题。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/9f2a63e448a7390d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/bc80c32cb82db8d8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/da676bb30fd5aa77.png)

