---
title: LLM 输出解析的防御性编程：JSON 与标签格式混合处理
feedId: 35066
source: 综合讨论
publishedAt: 2026-08-28
---

## 背景

在 OpenClaw / Agent / MCP 插件链路里，我们经常把大模型输出当作“结构化数据”直接 `json.loads`。模型却会返回这些形态：

- ` ```json\n{...}\n``` `
- `这里是结果：\n{...}`
- `<response>{...}</response>`
- `{...}\n{...}` 多个 JSON
- JSON 前后混着解释文本

如果解析器只看 happy path，一旦上游模型版本、温度、Prompt 小改，就会在 pipeline 中段抛异常。防御性解析不是“万能正则”，而是把解析变成候选提取 + 分级尝试。

## 问题

脆断式解析有两个典型痛点：

1. 直接 `json.loads` 只能处理纯 JSON，遇到代码块、标签、BOM、首尾空白变体就挂。
2. 一个正则 `re.search(r'\{.*\}', text, re.S)` 贪婪匹配会把多个对象并成一个，或者被字符串里的 `}` 截断。

实际工程里更麻烦的是“标签 + JSON”混合：模型有时输出 `<json>{"a":1}</json>`，有时把 `<` 当 HTML 实体写成 `&lt;json&gt;`，有时标签带属性 `<response type="json">`。这些不能靠单一 `strip` 解决。

## 做法/步骤

建议把解析拆成四层，而不是一个函数直接出结果。

### 1. 归一化输入

先去掉 BOM、`\r`、首尾无意义空白。不要在这层删除任何文本内容，因为模型可能在字符串字段里包含换行或空白。

```python
def normalize(text: str) -> str:
    text = text.lstrip("\ufeff")
    text = text.replace("\r\n", "\n").replace("\r", "\n")
    return text.strip()
```

### 2. 提取候选 JSON 片段

候选提取按优先级来，不是一次性截断：

1. 如果存在 Markdown 代码块围栏，优先取围栏内容；多个围栏时逐个尝试。
2. 如果存在 `<json>`、`<response>` 这类包裹标签，优先取标签内的文本。
3. 定位第一个 `{` 或 `[`，用状态机做括号配平，取完整片段；没配平就放弃。
4. 如果出现多个 `{...}` 片段，保留所有候选，后续选择一个能解析且字段匹配度最高的。

括号配平必须处理字符串和转义，不能只数 `{` 和 `}`：

```python
def find_balanced_json(text: str, start: int):
    stack = []
    in_str = False
    escape = False
    for i in range(start, len(text)):
        ch = text[i]
        if in_str:
            if escape:
                escape = False
            elif ch == "\\":
                escape = True
            elif ch == '"':
                in_str = False
            continue
        if ch == '"':
            in_str = True
            continue
        if ch in "{[":
            stack.append(ch)
        elif ch in "}]":
            if not stack:
                return None
            if (ch == "}" and stack[-1] != "{") or (ch == "]" and stack[-1] != "["):
                return None
            stack.pop()
            if not stack:
                return text[start:i+1]
    return None
```

### 3. 标签剥离与 HTML 实体处理

针对标签混合，不用宽泛正则去删“所有尖括号”，而是先识别已知标签，例如 `json`、`response`、`result`。用非贪婪匹配拿到标签内文本后，再丢回候选提取。对 `&lt;`、`&gt;`、`&quot;` 只在确认是 XML/HTML 包裹层时做一次反转义，不要全局 `html.unescape`，否则字段里的 `&amp;` 会被改坏。

```python
import re, html

TAG_RE = re.compile(r"<\s*(json|response|result)\s*[^>]*>(.*?)<\s*/\s*\1\s*>", re.I | re.S)
m = TAG_RE.search(text)
if m:
    candidates.append(m.group(2))
```

### 4. 分级解析

每个候选片段依次尝试：

1. `json.loads`，`strict=False`，允许控制字符。
2. `json5.loads` 或兼容库，处理尾部逗号、注释。
3. 对候选做最小修复：去掉尾部逗号、把单引号替换成双引号。
4. 对多个候选，解析成功后再按必需字段过滤，选最完整对象。

关键点：不要把“模糊解析成功”直接当作最终结果，除非它满足 schema/必需字段。宁可输出解析失败日志和 raw text，让上层重试或降级，不要静默返回错误 dict。

## 踩坑点

- 贪婪正则 `\{.*\}` 会跨对象：两个 JSON 直接粘在一起时，它会把中间和其他内容全吞掉。
- 字符串里包含 `}`：如果不处理字符串状态，括号计数法会在字段内容处提前闭合。
- 代码块围栏内还可能再出现 ` ``` `：例如模型输出 code fence 内嵌示例，不能只按第一个三反引号截断。
- `json.loads` 对 `NaN`/`Infinity` 默认会抛错。需要设置 `parse_constant` 转成 `None`，或者交给 json5。
- 遇到 `&quot;` 这类 HTML 实体，先不要全局反转义，否则把 JSON 字符串里的真实 `&quot;` 改成 `"`，破坏语义。
- 标签属性 `<response type="json">` 容易被简单 `strip("<response>")` 漏掉或误伤。
- 不要删除所有非 JSON 文本再解析：可能把模型输出里的 key 或 value 删掉。

## 可复用建议

1. 把候选提取做成无状态小工具，例如 `extract_json_candidates(text, tags=("json","response","result"))`，后续 Agent 工具调用统一复用。
2. 为每个失败 case 保留原始输出和候选列表，打成结构化日志。没有 raw 输出，回归排查成本很高。
3. 解析函数返回 `(parsed, meta)`，meta 里标注用了哪一层候选、是否经过标签剥离、是否用 json5，便于观测模型输出漂移。
4. 用固定 bad cases 做 unit test：代码块围栏、多 JSON、`<json>` 包裹、`&quot;` 实体、尾部逗号、注释、字符串内 `}`、前后解释文本。
5. 如果某个 Agent 的输出协议能改，优先在 Prompt 里约定“只输出一个 JSON 对象，禁止代码块和解释文本”；但防御性解析仍作为第二道防线。

## 总结

LLM 输出解析的防御性编程，目标不是解析所有奇怪格式，而是把“不确定输入”变成可分级、可观测、可回退的流程。JSON 和标签混合时，先归一化、按优先级提取候选、做括号配平、再分级解析，通常比单步正则或脆断 `json.loads` 稳得多。对 Agent / MCP / 插件自动化来说，这类小工具值得沉淀到基础库，因为模型输出不会一直符合你的预期。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/80d3dde99f81cb0b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/4bad8ba7af0f7f16.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/2cca5217b0daef2d.png)

