---
title: LLM 输出解析的防御性编程：JSON 与标签混合格式处理实战
feedId: 32504
source: 综合讨论
publishedAt: 2026-08-11
---

在基于大语言模型的自动化链路（Agent、MCP 工具、OpenClaw 插件）中，我们经常要求模型返回结构化数据——JSON 是最常见的选择。实际情况是，即使 prompt 明确要求“只输出 JSON，不要任何解释”，模型仍可能输出带 markdown 代码块包裹、前导文字、甚至 JSON 片段混在自然语言中的内容。如果不做防御性解析，下游流程就会因为一个意外的 \`\`\`json 而全线崩溃。

本文整理一套在工程实践中反复验证过的防御性 JSON 解析策略，专门处理“标签 + JSON”混合输出，适合在工具调用、记忆提取、状态机跳转等强依赖结构化结果的场景中使用。

## 问题场景

典型的“非标”输出模式：

- **Markdown 包裹**：  
  \`\`\`json\n{"action": "read", "path": "/data"}\n\`\`\`  
  或 \`\`\`\n{...}\n\`\`\` 甚至没有语言标识。

- **前后缀文本**：  
  “这是你需要的数据：\n{"status": "ok"} \n 如果有什么问题请告诉我。”

- **多 JSON 片段**：  
  模型在一个回复中输出了两段独立 JSON，可能分别代表不同含义。

- **截断的 JSON**：  
  因为 token 限制，输出在中间断开，例如 \`{"items": [1, 2,\` 缺失闭合括号。

- **格式瑕疵**：  
  末尾多余逗号、使用单引号、包含 JavaScript 风格注释、数字字段用字段名未加引号等。

如果解析器只是简单 `json.loads(response)`，几乎必然在生产中抛异常。我们需要的是一个能自动“清洁”、定位、修复并返回可用对象的防御层。

## 防御性解析管线设计

核心思想：不信任任何输出格式，分层尝试恢复出合法 JSON。流程如下：

### 1. 文本预处理：剥离外层包装

先用正则移除 markdown 代码块标记，保留内部内容。关键要处理可能的变体：

- 三种引号：\`\`\`json、\`\`\`、\`\`\`\` （4 个反引号在某些平台出现）
- 前缀空格、换行与后缀空白
- 代码块后可能紧跟其他文本，需选取第一个完整块

示例策略：

```python
import re

def strip_markdown(text: str) -> str:
    # 尝试匹配 ```language ... ``` 或简单的 ```...```
    pattern = r"```(?:json|JSON)?\s*\n?(.*?)```"
    match = re.search(pattern, text, re.DOTALL)
    if match:
        return match.group(1).strip()
    return text
```

如果不存在代码块，直接返回原文供下一步处理。

### 2. 快速定位 JSON 边界

很多响应中 JSON 嵌在自然语言里。一个健壮的方法是查找最外层 `{` 和对应的 `}` 或 `[` 和 `]`。可以使用简单的括号计数来抽取首个完整对象/数组：

```python
def extract_first_json(text: str) -> str | None:
    starts = [{"open": "{", "close": "}"}, {"open": "[", "close": "]"}]
    for pair in starts:
        start = text.find(pair["open"])
        if start == -1:
            continue
        count = 0
        for i, ch in enumerate(text[start:], start=start):
            if ch == pair["open"]:
                count += 1
            elif ch == pair["close"]:
                count -= 1
                if count == 0:
                    return text[start:i+1]
    return None
```

若找到候选 JSON，传给下一步解析；否则直接把整段清洗后的文本拿去尝试。

### 3. 初次严格解析

用 `json.loads` 解析候选字符串。如果成功则马上返回，因为这是最高效且最严格的路径。捕获 `JSONDecodeError` 后进入修复模式。

### 4. 容错修复

生产中最实用的方案是引入 `json5` 或 `pyjson5`，它们可以自动容忍尾逗号、单引号、注释等常见问题。如果仍失败，退一步使用 `demjson3`（支持更多修复策略，但较慢）。可组合使用：

```python
def robust_parse(json_str: str):
    try:
        return json.loads(json_str)
    except Exception:
        pass
    # 尝试 json5
    try:
        import json5
        return json5.loads(json_str)
    except Exception:
        pass
    # 最后尝试 demjson3（需安装）
    try:
        import demjson3
        return demjson3.decode(json_str)
    except Exception:
        pass
    return None
```

### 5. 处理截断与修复

对于被截断的 JSON，一个简单但有效的做法是尝试补全缺失的括号。例如从字符串末尾反向扫描，补上缺失的 `}` 或 `]`，再次解析。但需要小心，补错括号可能导致错误的复原。通常只对明确由 token 限制导致的截断做修复，并记录 warning 日志。

如果截断导致无法修复，可考虑只返回已解析的部分数据（如 `items` 数组中的部分元素），并在返回结果中附带 `truncated: true` 的标志，供上层逻辑决定是否重试或降级。

### 6. 多片段处理

对预期可能输出多个 JSON 的场景（例如“每次操作输出一句 JSON”），可以循环调用 `extract_first_json`，每次将已抽取部分从文本中移除，直至无剩余合法 JSON。然后将它们收集成列表返回。这种做法在批量工具调用场景中非常有用。

## 踩坑记录

- **代码块中嵌套代码块**：如果模型在 JSON 内包含示例 markdown，可能出现 ``` 被误判为结束标记。此时正则应当取最外层，所以采用最短匹配 `(.*?)` 多数情况可行，但极端时需用更精确的括号平衡方案，而不是仅依赖反引号。
- **Unicode 转义与换行**：模型输出的 JSON 字符串内可能包含 `\n`、`\u` 序列，直接 load 通常没问题，但如果混入了未转义的真实换行，要先清洗，否则 `json.loads` 会报控制字符错误。可以在解析前将文本中的不可见控制字符（换行、制表除外）转义。
- **模型输出“思考过程”**：部分模型（特别是 o1 类）会在 JSON 前输出推理过程，且无法通过 prompt 完全抑制。我们的 `extract_first_json` 可以应对这种情况，但要注意推理文本本身可能包含花括号，需要选择最外层完整的 JSON 对象，而非被假括号蒙骗。
- **浮点数与特殊值**：`NaN`、`Infinity` 不是合法 JSON，但某些模型会输出。在用 `json5` 时可解析，但标准 `json` 模块不行。统一用容错库可以避免此类异常。

## 可复用建议

封装一个 `defensive_json_parse(response: str, expect_type: str = "object")` 函数，参数控制预期类型（对象/数组），内置上述逻辑。同时记录原始响应到日志系统，这样即使解析失败也能回溯上下文，而不是对着空异常摸索。将该函数作为所有 LLM 调用出口的标准后处理器，有条件地开启 debug 模式可以将每一次解析尝试的过程记录下来，便于事后分析模型行为变化。

另外，谨慎使用正则去暴力清洗——永远优先通过结构定位（括号计数）。正则可以作为辅助（如剥离代码块），但不要用来匹配嵌套结构。

## 总结

让大模型输出结构化 JSON 是工程常态，但模型的输出不可控性决定了我们不能假设“格式永远正确”。设计一条分层防御的解析管线，按“剥离外层→定位边界→严格解析→容错修复→截断处理→多段提取”的顺序演进，能够大幅提升自动化链路的鲁棒性。这种防御性编程思维也同样适用于处理 XML、YAML 等任何预期结构化输出的场景。

将解析与日志、监控结合，才能真正把“偶然的格式瑕疵”变成“可预期、可分析的系统行为”。下次当你的 Agent 因为一个意外的 markdown 包裹而罢工时，不妨试试这套策略。

---

