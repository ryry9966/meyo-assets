---
title: 给 LLM 输出加一道“格式安全网”：JSON 与标签混合解析的防御性编程实践
feedId: 35318
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景

在 OpenClaw、Agent、MCP 插件或自动化流水线中，我们经常要求 LLM 返回结构化数据，比如一个 JSON 对象，用于触发下一步动作。理想情况下，输出是干净的：

```json
{"action": "send_email", "to": "user@example.com"}
```

但真实环境中，LLM 的输出往往“很有个性”：

- 有的包在 Markdown 代码围栏里：```` ```json ... ``` ````
- 有的先来一段解释，再给 JSON：“好的，结果如下：”
- 有的用自定义标签包裹：`<result>{"action": ...}</result>`
- 有的甚至混合：文本里嵌了 JSON，标签内又套了一层 JSON，或者标签是 `[RESULT]...[/RESULT]` 风格
- 还有的输出了 JavaScript 风格的对象（单引号、无引号键名）

如果下游直接 `json.loads()`，大概率会抛异常，导致整个自动化流程中断。重试可能有用，但更可控的方式是在入口处加一层防御性解析，把“格式抖动”消化掉。

## 问题

在 OpenClaw 插件开发中，我遇到过这样的输出：

```
好的，以下是根据你的请求生成的结果：
```json
{"action": "send_email", "to": "user@example.com", "subject": "Hello"}
```
如果需要修改，请告诉我。
```

直接 `json.loads` 失败，因为字符串包含中英文说明和代码围栏。另一个场景是 MCP 工具返回：

```
<result>{"status": "ok", "data": {"id": 123}}</result>
```

而本地解析代码只认裸 JSON，导致插件工作不稳定。这类问题不是模型能力不足，而是 LLM 对“纯输出格式”的遵循天然不稳定，尤其是当 prompt 较长、任务复杂或模型温度较高时。

## 做法/步骤

我封装了一个 `robust_parse_json` 函数，按以下层次依次尝试：

### 1. 剥离 Markdown 代码围栏

先用正则把 ```` ```json ... ``` ````、```` ``` ... ``` ```` 中的内容提取出来。注意语言标识可能是 `json`、`javascript`、`python` 甚至没有，围栏也可能不闭合。我会用非贪婪匹配，并允许可选的结束围栏缺失。

```python
import re

def strip_code_fence(text: str) -> str:
    # 匹配 ``` 开头，可选语言标识，然后内容，可选结尾 ```
    m = re.search(r"```[a-zA-Z]*\s*(.*?)```", text, re.DOTALL)
    if m:
        return m.group(1).strip()
    # 如果只有开头没有结尾，尝试截取开头之后内容
    m = re.search(r"```[a-zA-Z]*\s*(.*)", text, re.DOTALL)
    if m:
        return m.group(1).strip()
    return text
```

### 2. 扫描所有可能的 JSON 对象

不要用“从第一个 `{` 到最后一个 `}`”这种粗暴截取，因为字符串值里可能有大括号。使用 Python 标准库的 `json.JSONDecoder().raw_decode`，它可以从指定位置开始解析一个 JSON 对象，并返回对象和结束位置。我从字符串每个 `{` 位置尝试，记录成功解析为 `dict` 的候选。

```python
import json

def extract_json_objects(text: str):
    candidates = []
    decoder = json.JSONDecoder()
    idx = 0
    while idx < len(text):
        if text[idx] == '{':
            try:
                obj, end = decoder.raw_decode(text, idx)
                if isinstance(obj, dict):
                    candidates.append(obj)
                    idx = end
                    continue
            except json.JSONDecodeError:
                pass
        idx += 1
    return candidates
```

这样即使文本前后有说明，也能提取出所有 JSON 对象。

### 3. 提取标签内容

对于 `<result>...</result>` 或 `[RESULT]...[/RESULT]` 这类自定义标签，用非贪婪正则提取内部内容，然后对内部内容递归调用 `robust_parse_json`，因为里面可能又是 JSON 或围栏包裹的 JSON。

```python
TAG_PATTERNS = [
    r"<result[^>]*>(.*?)</result>",
    r"\[RESULT\](.*?)\[/RESULT\]",
    r"<output[^>]*>(.*?)</output>",
    r"\[OUTPUT\](.*?)\[/OUTPUT\]",
]

def extract_tag_content(text: str):
    for pat in TAG_PATTERNS:
        m = re.search(pat, text, re.DOTALL | re.IGNORECASE)
        if m:
            return m.group(1).strip()
    return None
```

### 4. 兜底：宽容解析与基本修复

如果自然提取失败，可以尝试一些保守修复：替换中文引号为英文、把单引号转为双引号、移除尾随逗号等。但这些操作有风险，可能改变数据语义。我只在确定是格式问题（比如输出整体明显是 JSON 但引号错误）时才用，并且会记录一条警告日志。

### 5. Schema 校验

解析出候选 dict 后，用 Pydantic 或简单的 `required_keys` 检查。如果有多个候选，选择第一个通过 schema 校验的；如果没有明确候选，取包含最多业务字段的那个。最后把原始输出和解析结果一起写入日志，方便回溯。

核心逻辑：

```python
def robust_parse_json(text: str, required_keys=None):
    original = text
    # 1. 代码围栏剥离
    text = strip_code_fence(text)
    # 2. 直接尝试解析整体
    try:
        obj = json.loads(text)
        if isinstance(obj, dict):
            return obj
    except:
        pass
    # 3. 扫描 JSON 对象
    candidates = extract_json_objects(text)
    # 4. 标签提取并递归
    tag_content = extract_tag_content(text)
    if tag_content:
        tag_candidates = extract_json_objects(tag_content)
        candidates.extend(tag_candidates)
        try:
            tag_obj = json.loads(tag_content)
            if isinstance(tag_obj, dict):
                candidates.append(tag_obj)
        except:
            pass
    # 5. schema 筛选
    if required_keys:
        for cand in candidates:
            if all(k in cand for k in required_keys):
                return cand
    # 6. 返回第一个 dict 候选，或 None
    return candidates[0] if candidates else None
```

## 踩坑点

- **非贪婪匹配陷阱**：如果 JSON 字符串值里含 `</result>` 或 `}`，简单正则 `.*?` 可能提前截断。要使用 `raw_decode` 或平衡括号计数，而不是正则。
- **多个 JSON 对象**：LLM 有时返回多个候选，比如先给一个示例再给真实结果。需要用 schema 区分，不要盲目选第一个。
- **代码围栏不闭合**：有的模型只写了开头 ```` ```json ```` 但忘了结尾，`strip_code_fence` 要兼容。
- **标签大小写和属性**：`<RESULT>` vs `<result id="1">`，正则要加 `IGNORECASE` 并允许属性。
- **不要过度修复**：自动补全括号、添加缺失逗号等可能把错误数据变成“看似合法”的数据。宁可失败并让上层重试，也不要静默生成错误结果。
- **日志记录**：一定要保留原始输出，调试时能看到模型到底输出了什么，而不是只看到解析失败堆栈。

## 可复用建议

1. **封装成独立模块**：把 `robust_parse_json` 放到插件公共库，所有接收 LLM 输出的地方统一调用，避免每个工具重复造轮子。
2. **结合重试机制**：如果解析失败，可以在 OpenClaw 流程里让 LLM 重新生成一次，提示“请只返回 JSON，不要包含其他文字”。重试 1-2 次即可，不要无限循环。
3. **优先使用 function calling / tool schema**：如果模型支持工具调用或 JSON mode，尽量用平台能力约束格式，防御性解析只作为兜底。
4. **写单元测试**：把常见的输出变体（围栏、标签、混合文本、多 JSON、错误引号）都写成测试用例。每次修改解析逻辑后跑一遍，防止回归。
5. **收集真实失败样本**：在生产日志中定期检查解析失败的原始输出，提炼新的模式加入测试集，持续增强鲁棒性。

## 总结

LLM 的格式不稳定是工程现实，不能靠 prompt 一句话就完全解决。在 OpenClaw、Agent、MCP 插件这类自动化场景中，建立一个“宽容解析、严格校验”的中间层，能显著降低流程因格式抖动而中断的概率。防御性编程不是追求完美解析所有可能输出，而是用可控的成本兜住大多数常见变体，同时通过日志和测试把未知情况暴露出来，逐步迭代。最终目标：让业务代码只面对干净、已校验的 dict，而不是五花八门的原始字符串。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/444fc0d29ab27a44.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/f1ad901e68833ae2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/bfc0c2eecac4c5d4.png)

