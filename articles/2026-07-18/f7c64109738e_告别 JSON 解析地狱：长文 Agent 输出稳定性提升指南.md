---
title: 告别 JSON 解析地狱：长文 Agent 输出稳定性提升指南
feedId: 29461
source: 综合讨论
publishedAt: 2026-07-18
---

## 背景：JSON 不是万能格式

在 OpenClaw 这类 Agent 框架里，我们习惯用 JSON 作为工具调用和结构化输出的标准格式。模型输出 JSON → 解析 → 字段映射，流程丝滑。但当任务从“返回一句话”变成“生成一篇包含多段分析、代码、列表的长文”时，JSON 开始频频翻车。

最常见的问题：
- 模型生成的长文本中包含双引号、换行符，破坏了 JSON 结构；
- 模型在输出到一半时被截断（如 token 限制或流式中断），整个 JSON 对象不完整；
- 多层嵌套 JSON 导致解析器崩溃，或者模型自己把花括号搞丢。

我们团队在一个自动化报告生成场景中（MCP + OpenClaw）经历了连续数周的 JSON 修复噩梦。最后决定抛弃 JSON，转用**自定义标签格式**，长文输出的稳定性直接提升了 90% 以上。

## 问题拆解：为什么长 JSON 容易坏

JSON 对边界敏感。一个漏掉的花括号、一个错误的转义，整个解析失败。在长文本生成中，模型需要同时关注内容语义和格式语法，注意力分散，错误率会随长度指数上升。

更致命的是，Agent 工作流中常常需要**分块输出、流式解析**。JSON 作为树状结构，必须在完整接收后才能解析。如果有一次流式中断，前面的内容全部丢弃，无法增量提取。

标签格式的优势恰在于：
- 线性结构，不依赖配对符号（花括号），容易正则匹配；
- 允许部分提取，即使后面截断，前面已闭合的标签内容仍可正常使用；
- 对模型更友好，类似 Markdown/HTML 的起止标记，不要求深层嵌套。

## 做法：设计可解析的标签体系

我们没有直接用 XML/HTML，因为模型在生成代码时易混淆。设计了一组简单标签，例如：

```
<SECTION type="analysis">
内容...
</SECTION>
<SECTION type="code" language="python">
代码块...
</SECTION>
```

规则：
1. 每个区块用 `<SECTION>` 标签包裹，用属性 `type` 区分内容类型。
2. 标签内的文本不需要任何转义（仅禁止出现 `</SECTION>` 本身，极少数情况可替换为 `<\\/SECTION>`）。
3. 支持嵌套？**尽量避免**。如果真的需要层级，改用 `level` 属性，如 `<SECTION type="subsection" level="2">`。
4. 要求模型输出时，在 system prompt 给出明确约定：必须使用上述标签，并保证每个 `SECTION` 正确闭合。

解析端直接使用正则提取所有 `<SECTION type="...">` 到 `</SECTION>` 块，用 `re.DOTALL` 匹配跨行内容。即使整个输出被截断，也能拿到最后一个闭合标签之前的全部内容。

在 OpenClaw 中实现的一个典型流程：

```python
import re

def parse_sections(text):
    pattern = r'<SECTION\s+type="(?P<type>[^"]+)"(?:\s+level="(?P<level>\d+)")?\s*>(?P<content>.*?)</SECTION>'
    matches = re.finditer(pattern, text, re.DOTALL)
    sections = []
    for m in matches:
        sections.append({
            "type": m.group("type"),
            "level": m.group("level") or "1",
            "content": m.group("content").strip()
        })
    return sections
```

然后按 type 分流：分析段落直接拼入 Markdown，代码块做语法高亮，表格数据可二次解析。整个流程不再依赖一个合法 JSON，心智负担骤降。

## 踩坑记录

1. **标签名与内容冲突**  
   一段代码或文本中恰好出现 `<SECTION>` 字符串，会导致错误匹配。解决方式：在 prompt 中要求模型如果需要在内容中展示类似标签，使用 `<[SECTION]>` 或加空格，并在解析前用简单规则过滤无意义的干扰。

2. **不闭合的标签导致漏数据**  
   虽然情况较少，但模型偶尔会漏掉 `</SECTION>`。我们在解析时增加一个容错兜底：如果遇到 `<SECTION` 但其后没有找到对应闭合，就认为后面的所有内容都属于该类型，取到文末。虽不完美，但比丢弃整段好。

3. **过度嵌套**  
   最初设计的标签天然鼓励嵌套，结果模型开始搞出 `<SECTION type="outer"><SECTION type="inner">` 这种。正则无法正确处理。后来强制规定不嵌套，改为用 `level` 属性，结构扁平化。

4. **模型“创造性”修改标签**  
   虽然 prompt 约束了标签名，模型有时仍会自己发明 `<SECTION type="summary">` 或改成 `type='analysis'`（单引号）。解决办法：在 system prompt 里增加少量 few-shot 示例，强调格式精确性，并配合解析后的 type 白名单过滤未知类型。

5. **流式场景下的提前解析**  
   流式输出时，每收到一块数据就合并缓冲。只有当遇到 `</SECTION>` 才认为一个完整区块生成，可以解析。我们在 OpenClaw 的 streaming hook 里做了简单的状态机，避免解析未闭合的片段。

## 可复用建议

- **从简开始**：一个 `<SECTION type="xxx">` 足够应付 90% 的长文结构化需求。不要提前设计复杂标签语言。
- **坚守 flat 结构**：除非你有真实的树状需求并愿意写递归解析，否则用 `level` 属性模拟层级。
- **做足容错**：正则匹配不到时回退到原始文本输出，让下游能降级使用，别让整个流水线崩溃。
- **提示词工程不可省**：给模型足够的约束和示例，远比后处理修复有效。可以在 prompt 中告诉模型“如果需要在内容中展示标签，请转义为 <[TAG]>”。
- **监控与统计**：在线上统计标签闭合率、type 分布，快速定位模型行为漂移。

## 总结

长文结构化输出中，JSON 的闭合约束和转义规则成为稳定性的瓶颈。自定义标签格式凭借宽松的解析、对截断的容忍以及简单的匹配逻辑，在 Agent 生产环境中表现优异。它不追求数据结构的完备性，而是追求**工程可用的鲁棒性**。对于 OpenClaw/MCP 这类高度自动化的系统，这往往比理论上的“标准格式”更重要。

如果你正被坏 JSON 折磨，不妨让模型放弃花括号，试试标签。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-18/5fbafec97dce8059.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-18/d289675b6646512d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-18/bf1ae721cab88780.png)

