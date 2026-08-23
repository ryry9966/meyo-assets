---
title: LLM 输出解析的防御性编程：JSON 与标签格式混合处理
feedId: 34423
source: 综合讨论
publishedAt: 2026-08-24
---

## 背景

在 OpenClaw、Agent、MCP 插件这类链路里，我们经常让 LLM 返回结构化数据。约定是 JSON，但真实输出往往不是干净的 `application/json`，而是带 markdown 围栏、说明文字、甚至 XML 风格标签的混合文本。例如：

```text
好的，结果如下：
```json
{"tool": "search", "args": {"q": "防御性解析"}}
```
希望对你有帮助。
```

或者：

```text
<result>{"status": "ok", "data": [1, 2, 3]}</result>
```

如果直接对整段文本调用 `json.loads`，大概率会在第一个字符就失败。随后 Agent 报错、流程中断，或者触发昂贵且不稳定的重试。这里需要的不是更严格的 prompt，而是一个防御性解析层：先尽可能从混合文本中提取出合法 JSON，再进行校验。

## 问题

常见的失败形态包括：

- ` ```json ... ``` ` 代码围栏，前后夹带自然语言
- `<output>`、`<result>`、`<json>` 等标签包裹
- JSON 前有 BOM 或零宽字符
- 字符串值中包含换行、嵌套引号或反斜杠，正则贪婪匹配直接截断
- 多个 JSON 对象连续出现，解析器只取第一个
- 模型输出了 JSON5 注释或单引号，标准 `json` 不认

这些问题单独看都不难，但混在一起时，一次性写出“稳”的解析器就有难度。

## 做法

核心思路是：**不要直接 `json.loads`，先提取候选片段，再逐个尝试解析**。以 Python 为例，一个起步版本可以这样组织：

```python
import json
import re

TAG_RE = re.compile(
    r"<(?:output|result|json|response)\b[^>]*>(.*?)</(?:output|result|json|response)>",
    re.I | re.S,
)

def extract_json(text: str):
    candidates = []

    # 1. 提取 ```json ... ``` 或 ``` ... ```
    fence = re.search(r"```(?:json)?\s*(.*?)```", text, re.S | re.I)
    if fence:
        candidates.append(fence.group(1))

    # 2. 提取已知标签包裹的内容
    candidates.extend(TAG_RE.findall(text))

    # 3. 原始 JSON 扫描：从每个 { 或 [ 开始尝试解析
    decoder = json.JSONDecoder()
    idx = 0
    while idx < len(text):
        if text[idx] in "{[":
            try:
                obj, end = decoder.raw_decode(text, idx)
                candidates.append(text[idx:end])
                idx = end
                continue
            except json.JSONDecodeError:
                pass
        idx += 1

    # 4. 依次解析，成功即返回
    for cand in candidates:
        try:
            return json.loads(cand)
        except json.JSONDecodeError:
            continue
    return None
```

这个版本只处理了围栏、少量标签和原始扫描。实际落地时还需要加几件事：

1. **清洗零宽字符和 BOM**：`text = text.lstrip("\ufeff")`，并移除 `\u200b` 等常见干扰。
2. **对标签内容递归提取**：标签内可能还有围栏或嵌套标签，解析前先递归调用提取函数。
3. **支持多对象**：如果业务上可能返回多个 JSON 对象，可以收集所有 `raw_decode` 成功的片段，而不是在第一个成功时返回。
4. **schema 校验后置**：解析成功只是第一步，字段缺失、类型错误应在下一层用 Pydantic 或 JSON Schema 校验。
5. **必要时引入 json5 或 json_repair**：如果模型经常输出注释或单引号，可以在自定义提取之后、标准 `json.loads` 之前加一层修复；但不要完全依赖这些库处理混合格式，它们对围栏和标签的识别并不总是可靠。

## 踩坑点

- **不要用 `\{.*\}` 这类贪婪正则**：遇到嵌套对象或字符串里的大括号会截断错误。
- **不要全局替换全角引号**：比如把 `“` 替换成 `"`，可能误伤字符串内容。最好只在确定不是字符串值的区域做清洗，或者先提取再处理。
- **`raw_decode` 要从多个位置尝试**：第一个 `{` 可能是自然语言里的左花括号，真正 JSON 在更后面。上面的循环已经覆盖，但如果性能敏感，可以限制扫描长度。
- **标签正则不能写得太死**：模型可能会输出 `<result type="json">` 或 `<RESULT>`，需要忽略大小写并允许属性。
- **空结果要有兜底**：不要把解析失败直接抛给用户，记录原始输出并返回安全默认值或触发一次定向重试。
- **标准 `json` 不支持 `true/false/null` 以外的 Python 字面量**：不要用 `ast.literal_eval` 代替，容易在 `null`、`true` 上翻车。

## 可复用建议

- 把提取逻辑封装成独立工具函数，放在 MCP 服务或插件公共层，所有“LLM 返回 JSON”的入口统一调用。
- 在 prompt 里约定输出格式可以降低解析难度，但不要作为唯一保障；防御层仍然保留。
- 解析失败时记录原始输出和错误原因，便于积累回归样本。
- 单元测试用真实失败样例驱动，每次遇到新形态就补一条用例。
- 如果流程允许，解析失败后做一次“请严格按 JSON 格式重新输出”的定向重试，并设置最大次数。
- 最终数据必须经过 schema 校验，解析成功不等于数据可用。

## 总结

LLM 输出解析不是一个“用 `json.loads` 就能解决”的问题。工程上更合理的做法是：提取优先、解析在后、校验兜底。先把 JSON 从围栏、标签和自然语言中尽可能完整地剥离出来，再做结构化验证。这样不仅能减少 Agent 链路的中断，也能让后续的 MCP 插件和自动化任务更稳。防御性解析的成本不高，但收益会随着模型输出形态变多而持续体现。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/7e657ce79cf9c113.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/012d22be830ca156.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/81f2b58b281e6a41.png)

