---
title: LLM 输出解析的防御性编程：从容处理 JSON 与 Markdown 代码块混合
feedId: 31959
source: 综合讨论
publishedAt: 2026-08-07
---

## 背景：为什么“万能 JSON 解析”是个伪命题

在构建 OpenClaw、Agent、MCP 工具或插件时，我们经常依赖 LLM 返回结构化数据。你可能会在 prompt 中要求“输出 JSON”，但你很快会发现，模型的实际输出非常“自由”。有的模型老老实实返回纯 JSON，有的会包裹在 ```` ```json ```` 标记内，有些还会在前面加一段解释：“Sure, here is the JSON:”。如果直接 `json.loads`，你的 Agent 还没开始干活就崩了。

做过几个项目的人都知道，依赖 LLM 的输出格式就像依赖天气——永远都要带伞。我们需要一把好的“伞”：一个能容错、能修复、能应对 Markdown/JSON 混合输入的防御性解析器。

---

## 问题拆解：LLM 输出到底有多“脏”

实际遇到的典型情况包括：

1. **Markdown 代码块包裹**  
   ```` ```json\n{"key":"value"}\n``` ````
2. **代码语言标记不固定**  
   ```` ```javascript\n{"key":"value"}\n``` ```` 或者索性没有标记。
3. **前后缀文字**  
   “以下是结果：`{"name":"test"}`谢谢。”
4. **JSON 内部错误**  
   末尾多余逗号、缺少引号、单引号代替双引号等。
5. **多个 JSON 片段**  
   比如模型输出了两个对象：`{"a":1} {"b":2}`，但其实你只期望一个。

我们的解析策略必须足够宽容，又能保证语义正确性。

---

## 做法与步骤：一个实用的防御性解析函数

我推荐的核心思路是：**先提取，再修复，最后校验**。下面是一个 Python 实现，可以直接放进你的工具库。

### 1. 提取候选 JSON 字符串

先从原始文本中捞出最像 JSON 的部分：

```python
import re
import json
from json_repair import repair_json

def extract_json_candidates(text: str) -> list[str]:
    # 1. 尝试提取 markdown 代码块内容
    pattern = r"```(?:json|javascript|js)?\s*\n?(.*?)\n?```"
    matches = re.findall(pattern, text, re.DOTALL | re.IGNORECASE)
    if matches:
        return [m.strip() for m in matches]

    # 2. 如果没有代码块，尝试找到第一个 { 或 [ 到最后一个 } 或 ]
    # 简单起见，直接用平衡括号截取（更复杂的生产实现可以用栈）
    brace_start = re.search(r'\{', text)
    brace_end = re.search(r'\}', text[::-1])
    if brace_start and brace_end:
        start = brace_start.start()
        # re.search on reversed string gives end index from end
        end = len(text) - brace_end.start()
        candidate = text[start:end].strip()
        return [candidate]

    return []
```

> 实际代码中，最好加上对 `[` 和 `]` 的支持，这里篇幅所限只展示思路。

### 2. 逐个尝试解析，失败则修复

```python
def parse_llm_json(text: str):
    candidates = extract_json_candidates(text)
    last_error = None
    for candidate in candidates:
        try:
            # 优先尝试标准库直接加载
            return json.loads(candidate)
        except json.JSONDecodeError as e:
            last_error = e
            # 用 json_repair 修复常见错误
            try:
                repaired = repair_json(candidate)
                return json.loads(repaired)
            except Exception:
                continue
    # 所有候选者都失败，抛出明确的异常供上层处理
    raise ValueError(f"Cannot parse JSON from LLM output. "
                     f"Raw text: {text[:200]}... | Last error: {last_error}")
```

这里用到了 `json_repair` 库（`pip install json-repair`），它能自动补全缺失的引号、删除多余逗号、处理单引号等问题，对 LLM 输出的“小毛病”效果很好。

### 3. 可选的 Schema 校验

解析成功后，强烈建议做一层业务校验，例如用 Pydantic 验证字段类型和必填项。如果校验失败，可以触发重试逻辑，而不是任由脏数据进入下游。

---

## 踩坑点实录

- **代码块内有多个 JSON**：如果 `extract_json_candidates` 用 `re.findall` 会返回多个匹配，但那是**多个代码块**。同一个代码块内可能出现 `{...} {...}`，这时我们的“平衡括号截取”可能会拿到一大段，里面还是两个对象。实际可先取第一个完整对象，用迭代的方式：找到第一个 `{`，用计数器找到对应的 `}`，截取它。
- **零宽字符或 BOM**：某些模型输出开头会有 `\ufeff`，直接用 `strip` 不管用。可以先 `text = text.encode('utf-8').decode('utf-8-sig')` 去掉 BOM。
- **浮点数 / 科学计数法**：一些模型会输出 `1e5`，标准 JSON 不支持，但 `json_repair` 可以处理。如果不用 repair，这类问题会直接报错。
- **非 JSON 结构的合法 Markdown**：比如模型在 ` ``` ` 内返回了一个 YAML 或纯文本，你的代码需要能识别并快速失败，不要强行当做 JSON 解析。
- **“防御”不要变成“纵容”**：过度宽容会让下游处理更困难。比如解析出字符串 `"value"`，但实际期望对象 `{"key":"value"}`。此时尽早暴露问题比默默接受更好。我们的原则是：**在解析层尽量宽容，在 Schema 校验层保持严格。**

---

## 可复用建议

1. **封装为标准函数**：在你的 Agent 工具集、MCP Server 或插件框架里，将上述逻辑封装成 `safe_json_parse()`，并全局复用。
2. **与 Output Parser 集成**：如果你在用 LangChain 或类似框架，可以自定义一个 `DefensiveJsonOutputParser`，内部使用上述提取+修复逻辑。
3. **记录失败样例**：在生产环境中，解析失败时除了抛异常，最好把原始输出截取前 500 字符写入日志。这些样本可以反哺 Prompt 优化。
4. **考虑异步与性能**：解析过程本身是同步且轻量的，不要引入不必要的异步开销。json_repair 对中等大小的 JSON（几十 KB）性能足够。
5. **不要依赖 LLM 的“自我纠正”**：可以让模型在失败时重试，但解析器本身必须稳健。两者结合才能真正降低错误率。

---

## 总结

只要你的系统里有一条 LLM 调用返回 JSON，防御性解析就是必选项。混合格式的处理不复杂，但细节不少。把“提取-修复-校验”这一套流程沉淀成工具函数，可以显著降低 Agent 或插件的脆性，让你的自动化流水线在数百次调用中依然安稳运行。

别信任任何模型的输出了——带上你亲手写的 JSON 雨伞，雨天也能跑完流程。

---

