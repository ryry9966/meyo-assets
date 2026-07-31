---
title: LLM 输出解析防御指南：如何处理 JSON 与标签格式的混合乱象？
feedId: 31136
source: 综合讨论
publishedAt: 2026-08-01
---

## 背景：当 Agent 不再能“相信” LLM 的输出

如果你正在用 OpenClaw 搭建工具链、定制 Agent，或者通过 MCP 把 LLM 接进自动化流程，那么你一定遇到过这种情况：要求模型返回干净的 JSON，但它偏偏给你带上一段开场白，或者把 JSON 包在 Markdown 代码块里，末尾还要加句“希望对你有帮助！”——更离谱时，JSON 里还会混入 `// 这是注释`，或者最后多一个逗号。

这些都不是“模型质量差”，而是当前大语言模型的输出本质上是一段未经过滤的自然语言采样。我们在工程里把 LLM 当作可调用的 API，就必须用防御性编程的思路来处理它的输出，而不是假设它永远严格遵守 prompt 指令。

## 问题：混合输出究竟长什么样？

在 OpenClaw 的任务编排中，一个典型的故障场景是：你配置了一个步骤，调用 LLM 提取结构化数据，期望得到的是一段合法的 JSON，结果实际返回的内容如下：

```
Sure, here is the result:
```json
{
  "name": "openclaw-demo",
  "version": "1.0.0",
  "dependencies": ["fastapi", "uvicorn",],
  // added for performance
  "optimization": true
}
```
Let me know if you need anything else.
```

对于标准的 `json.loads()` 来说，这根本就不是合法的 JSON。即便你让模型“只输出 JSON，不要 Markdown 标记，不要解释”，它也可能产生其他变形，比如：

- JSON 前后夹带自然语言解释
- 数组或对象的尾随逗号
- 用单引号而非双引号
- 布尔值写成 `True` / `False` 而不是小写
- 用 `null` 但偶尔写成了 `None`
- JSON 内部夹杂 `//` 或 `/* */` 注释
- 数字字段被写成字符串 `"123"` 或者反过来
- 多层嵌套中某个字段缺失、多了额外字段

对于要驱动下一步工具调用的 Agent 来说，这些“小问题”会直接导致整个工作流中断。而简单的重试（retry）有时并不能解决，因为模型可能持续生成类似的不合规格式。

## 做法：多级降级 + 野路子清洗，构建容错解析器

我们需要一个能处理混合格式的解析器，思路是“先提取疑似 JSON 片段，再逐级修复”。下面是我在 OpenClaw 插件开发中实际使用的一套步骤，代码不长，但覆盖了绝大多数异常情况。

### 第一步：从文本中提取 JSON 候选

不要把整段文本直接扔给 JSON 解析器。先尝试检测 Markdown 代码块（` ```json ` 或 ` ``` `），如果存在，只取代码块内部的内容；否则就采用括号匹配法，查找第一个 `{` 到最后一个 `}` 或第一个 `[` 到最后一个 `]` 作为候选。这步可以用简单的字符串操作或正则完成。

```python
import re

def extract_json_candidate(text: str) -> str:
    # 1. 优先提取 markdown json 代码块
    match = re.search(r"```(?:json)?\s*\n(.*?)\n```", text, re.DOTALL)
    if match:
        return match.group(1).strip()
    # 2. 否则，抓取第一个 JSON 对象或数组
    for pair in [("{", "}"), ("[", "]")]:
        start = text.find(pair[0])
        if start != -1:
            end = text.rfind(pair[1])
            if end != -1 and end > start:
                return text[start:end+1]
    return text
```

### 第二步：逐级修复，提高 JSON 合法性

拿到候选字符串后，不要直接 `json.loads`，而是尝试一系列修复。建议按以下顺序进行，每次修复后都尝试解析，一旦成功就返回：

1. **移除尾随逗号**：`re.sub(r",\s*([}\]])", r"\1", s)`
2. **将单引号转换为双引号**（但要注意不破坏字符串内部的单引号，较复杂，可用 `json.dumps` 间接处理或使用 `ast.literal_eval` 的容错逻辑）。一个折中方案是仅在键、值的外围尝试替换，但也要小心转义。最安全的方法是用正则匹配 `'(?![a-zA-Z])` 等，但工程中我常直接用 `demjson3` 这样的宽容解析库，或者先做 `eval()` 的尝试吗？绝对不要用 `eval`，安全风险太大。可使用 `json5` 或 `pyjson5` 库，它们原生支持注释、尾随逗号、单引号等。
3. **去除注释**：`re.sub(r"//.*?\n|/\*.*?\*/", "", s, flags=re.DOTALL)`
4. **处理布尔值大小写**：`s.replace("True", "true").replace("False", "false").replace("None", "null")`
5. **处理对象键的引号缺失**：有些模型会输出 `{name: "openclaw"}`，需要给未引号的键加上双引号。可使用正则 `re.sub(r"(\w+)\s*:", r'"\1":', s)`，但注意不要破坏已带引号的键。

### 第三步：结构化验证

即便通过了 `json.loads`，也建议对结果做 JSON Schema 校验。在 OpenClaw 的插件里，我习惯把期望的结构定义成 Pydantic 模型，解析成功后用 `Model.model_validate(obj)` 做二次校验。这可以捕获字段类型错误（例如插件期望 `port` 是 `int`，但模型返回了 `"8080"`），以及缺失必填字段等问题。

## 踩坑点：看似合规实则危险的细节

- **浮点数精度**：`json.loads("0.1")` 没问题，但模型可能输出 `1.` 这样的不合法数值。需要在修复时把 `1.` 替换为 `1.0`，否则 `json.loads` 会报错。
- **内嵌转义**：当 JSON 中包含用户输入的自然语言字符串，且模型没有正确转义换行符和引号时，会出现类似 `"message": "It's "ok""`，我们的单引转双引修复会雪上加霜。因此在移除注释、修复尾随逗号之后，最好先尝试用 `json5` 加载，它的容错能力更强。
- **多重代码块**：有的输出会在嵌套的 markdown 中再包含 JSON，如“为了清晰，我展示如下：```json ... ```”，并且还有外层解释。此时提取策略可能失败，需要从最内层代码块开始提取。
- **Agent 重试失效**：如果解析失败后简单触发 LLM 重试，且 prompt 不变，容易陷入死循环。建议在重试的提示词中**显式告诉模型上次输出的问题**，例如：“你的上一次输出包含了 Markdown 代码块标记和尾部多余文字。请只输出纯 JSON 对象，不要任何附加内容。”这能大幅提高修正成功率。

## 可复用建议：封装成 OpenClaw 工具函数

在 OpenClaw 的插件体系中，你可以将这个解析逻辑封装成一个可复用的工具函数 `robust_json_parse`，接受原始文本和可选的 Pydantic 模型，返回解析后的对象。同时记录解析失败的情况，便于后续调整 prompt 或回退到人工介入。

**关键代码骨架：**

```python
import json5
from pydantic import BaseModel, ValidationError

def robust_json_parse(text: str, model: BaseModel = None):
    candidate = extract_json_candidate(text)
    # 降级尝试：json5 直接解析
    try:
        obj = json5.loads(candidate)
    except Exception:
        # 进一步清洗后尝试标准 json
        cleaned = clean_common_issues(candidate)
        obj = json5.loads(cleaned)
    if model:
        return model.model_validate(obj)
    return obj
```

将这段逻辑内置到每个需要解析 LLM 输出的插件节点中，可以极大降低因格式问题导致的流程中断。

## 总结

LLM 输出的解析从来不是一条“解析成功或失败”的单一路径，而是一套需要层层降级、容忍常见变形的防御流程。在 OpenClaw 这样的 Agent 自动化框架里，花一点时间实现一个健壮的 JSON 提取与修复模块，换来的是整个工作流鲁棒性的指数级提升。不要相信模型会听话，用代码把所有的“不听话”路径都罩住，才是工程化落地该有的态度。

---

