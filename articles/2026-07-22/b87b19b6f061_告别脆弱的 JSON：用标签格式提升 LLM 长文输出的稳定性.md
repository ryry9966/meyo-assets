---
title: 告别脆弱的 JSON：用标签格式提升 LLM 长文输出的稳定性
feedId: 30065
source: 综合讨论
publishedAt: 2026-07-22
---

## 背景

在 OpenClaw 这类 Agent 自动化场景里，我们经常要求 LLM 输出长文本的结构化数据——比如会议纪要、分析报告、多步骤操作序列。最常见的做法是让模型返回 JSON，然后下游节点解析、提取字段。但**当输出超过几百行，JSON 就开始变得不可靠**：缺少闭合花括号、多余逗号、注释混入、引号未转义……这些错误会导致整个解析流程崩溃，甚至触发多次重试、浪费 token。

更麻烦的是，在流式输出场景中，我们可能拿到一份不完整的 JSON，根本没法在未收尾时就做渐进式处理。工程上需要一个更宽容、更易分段的解析方案。

## 问题

一个真实案例：用 GPT-4 生成一份“周报 + 待办清单 + 风险项”的结构化内容，prompt 要求返回这样的 JSON：

```json
{
  "summary": "...",
  "todos": [
    {"task": "...", "priority": "high"},
    ...
  ],
  "risks": "..."
}
```

结果是：大概每 10 次生成就有一次出现 JSON 解析错误。常见错误包括：

- 模型在 `risks` 字段中插入了一段带换行的自然语言，忘了转义引号。
- 流式输出被截断，缺少最后的 `}`。
- 模型“创造性”地添加了 JSON 不支持的注释 `// 这里需要注意`。

这些都不是“模型智商不够”，而是**长文输出的底层概率机制注定了格式会漂移**。JSON 对完整性和语法实在太敏感。

## 做法：切换到标签格式

我们的解决思路很简单：把 JSON 换成自定义的 XML 风格标签。例如：

```xml
<report>
  <summary>本周完成了……</summary>
  <todos>
    <task priority="high">修复登录超时</task>
    <task priority="medium">更新文档</task>
  </todos>
  <risks>后端服务偶发 502，需下周跟进。</risks>
</report>
```

在 prompt 中明确要求：只输出 `<report>…</report>` 包裹的内容，不要加 ```xml 标记或额外解释。这样有几个好处：

1. **容忍不完整**：即使响应被截断，我们可以提取最后一个完整的闭合标签，比如拿到了 `<summary>` 和部分 `<todos>`，仍然可以解析出已闭合的段落继续使用。
2. **无需转义**：标签内的纯文本大部分时间可以直接提取，除非内容包含 `<`，但我们可以通过要求模型“如果有 `<` 出现，替换为 `&lt;`”来规避（极少情况）。
3. **流式友好**：可以边接收边检查标签压栈，当见到 `</report>` 时即可认为输出完成，或利用计数器分段提取。

下面是一个最小可行的 Python 提取函数示例：

```python
import re

def extract_tag(content: str, tag: str) -> str|None:
    # 非贪婪匹配，允许换行，不要求内容转义
    pattern = rf"<{tag}>(.*?)</{tag}>"
    match = re.search(pattern, content, re.DOTALL)
    return match.group(1).strip() if match else None

def extract_report(content: str):
    summary = extract_tag(content, "summary")
    # 提取所有 task 标签内容
    tasks = [t.strip() for t in re.findall(r"<task[^>]*>(.*?)</task>", content, re.DOTALL)]
    risks = extract_tag(content, "risks")
    return {"summary": summary, "tasks": tasks, "risks": risks}
```

如果我们需要更完整解析整个 `<report>` 结构，也可以用 `extract_tag(content, "report")` 先得到文档体，再做二次提取，避免同名标签冲突。

## 踩坑点

**1. 同名标签嵌套**

比如存在 `<section><section>...</section></section>` 时，上面的非贪婪正则只能匹配到第一个闭合。对于这种情况，可以约定不使用同名嵌套，或者改用序号标签 `<section1>`、`<section2>`。如果项目确实需要嵌套，那就需要写一个简单的状态机栈解析。

**2. 模型输出多余空白或解释**

即便 prompt 要求“只输出标签内容”，模型偶尔还是会先来一句“好的，这是您的报告：”。我们会在解析前先定位第一个 `<report` 开始位置，把前缀裁掉，可以用 `content[content.find("<report"):]` 来粗切。

**3. 标签属性丢失**

上面的 `extract_tag` 不保存属性。对于 `<task priority="high">`，正则只捕获内容。如果需要属性，可以调整正则，捕获后手动解析，或要求模型不使用属性、改用子标签描述优先。

**4. 截断在标签名中间**

极端情况，比如输出末尾是 `<repo` 而不是 `<report`。此时所有正则匹配会认为该结构不完整而丢弃。建议在重试机制中，检测到解析失败时，将最后一个未闭合标签拼回下一次请求的后文，或直接要求模型从断点续写。

## 可复用建议

在 OpenClaw 的 Node 或 Plugin 中，可以封装一个 **“标签解析器”**，作为 JSON 解析的降级方案：

- 在 prompt 模板库中统一规范：用 `<output>` 包裹主体，内部定义稳定的标签名。
- 解析流程先尝试提取 `<output>` 整体，再分段提取；如果失败则回退到每项独立提取，保证至少取出部分字段。
- 结合重试策略：如果一次提取到的字段为空或关键字段缺失，把原始响应和错误位置一起发给模型进行一次“修复请求”，要求它“只输出修正后的 XML，不要解释”。
- 对于实时性要求高的 Agent，可通过比对 `</>` 的计数来判断是否还能继续接收，避免等到流超时才发现输出不完整。

这个方案还能很方便地和 MCP 工具结合：当需要模型返回需要工具确认的结构化参数时，标签格式比 JSON 更容易让中间审批节点插入旁注或修正，而不会破坏整体结构。

## 总结

把长文输出的结构载体从 JSON 换成自定义标签，并非否定 JSON 的优势，而是**在容错性和分段处理上做工程妥协**。标签格式让你能够“坏数据里捞出好部分”，而这恰恰是自动化和 Agent 链路中最需要的韧性。

在实际使用中，只需要一小段正则解析代码、加上对 prompt 的微调，就能显著降低因格式错误引起的重试次数。如果你的 OpenClaw 工作流因为长 JSON 输出反复崩溃，不妨花一小时试试标签方案——很可能就再也没有那群莫名其妙的 `JSONDecodeError` 了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/3254f9d5737a9957.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-22/d93a2fe3935bf201.png)

