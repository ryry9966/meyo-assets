---
title: LLM 输出解析的防御性编程：JSON 与标签的混合处理实战
feedId: 32323
source: 综合讨论
publishedAt: 2026-08-10
---

## 背景

在 OpenClaw 的 Agent、MCP 或插件开发流程中，我们经常要求 LLM 返回结构化数据（如 JSON 配置、指令序列或参数列表）。理想情况下，模型会输出干净的 JSON 字符串；但实际情况是，模型经常把 JSON 包裹在 Markdown 代码块中，或者在 JSON 前后附加解释性文字，甚至夹杂多语言注释、多对象输出。这会直接导致下游的 `json.loads()` 抛出异常，流程中断。

硬性要求模型“只返回 JSON”并不是工程上可靠的做法——模型会忘记指令，或受系统提示影响混入额外文本。我们需要一种**防御性解析策略**，让解析器像处理用户输入一样，容忍格式噪声，稳定提取可用的 JSON 对象。

## 典型问题

实践中常遇到的“脏 JSON”包括：

- 被 Markdown 代码块包裹：`` ```json { ... } ``` ``
- 前面有中文说明：“好的，这是结果：```json ... ``` 请查看。”
- JSON 内部含有多余的逗号、注释或字符串内的未转义引号（偶见于较弱模型）。
- 同时返回多个 JSON 对象（例如日志 + 结果对象）。
- 键名大小写不一致，或者字段缺失。

如果没有防御性处理，这些噪声会直接导致解析失败，在自动化流水线里就成了不稳定因素。

## 做法：三步解析 + 安全退化

我推荐一套**分层防御**的解析方案，按优先级处理：

### 第一层：智能剥离代码块

不要简单全局正则去匹配 `^```json`，因为可能多层嵌套。更稳健的方法是：

1. 先查找连续的代码块标记，提取最外层为 JSON 的块。
2. 如果同时存在多个代码块（可能一个是 JSON 示例，一个才是真实输出），优先选择**最后一个**标记为 `json` 或纯代码块，因为模型常把解释放前面、数据放后面。

```python
import re

def extract_json_block(text: str) -> str:
    # 匹配 ```json ... ``` 或 ``` ... ```，提取内容
    pattern = r'```(?:json)?\s*\n(.*?)\n```'
    matches = re.findall(pattern, text, re.DOTALL)
    if matches:
        return matches[-1].strip()   # 取最后一个代码块
    return text.strip()
```

退化策略：如果没有代码块标记，就把整个文本传递给下一层。

### 第二层：边界定位与容错解析

从剥离后的文本中，找到第一个 `{` 和最后一个 `}` 之间的内容，作为候选 JSON。这能处理文本前后残留字符的情况。然后尝试用 `json.loads()` 加载。

如果标准解析失败，不要立刻放弃。常见非标准格式需要**温和修复**：

- 末尾多余逗号：正则去掉对象/数组末尾逗号 `,(\s*[}\]])` → `$1`。
- 单引号替换为双引号（需小心字符串内部引号，不建议全局替换，最好用 AST 或只对键值分隔区域操作）。
- 移除行内注释 `//` 或 `/* */`（如果常见于模型输出）。

修复优先级：先尝试标准解析，失败后再逐步应用修复，每次修复后重新尝试，降低过度修改概率。

```python
import json

def safe_json_parse(text: str):
    # 尝试1：直接解析
    try:
        return json.loads(text)
    except json.JSONDecodeError:
        pass

    # 尝试2：去掉尾随逗号
    fixed = re.sub(r',\s*([}\]])', r'\1', text)
    try:
        return json.loads(fixed)
    except json.JSONDecodeError:
        pass

    # 尝试3：更激进地提取 {...} 内容
    match = re.search(r'\{.*\}', text, re.DOTALL)
    if match:
        try:
            return json.loads(match.group())
        except:
            pass
    return None
```

### 第三层：多对象分解

如果模型输出多个 JSON 对象（俗称“JSON lines”或拼接），我们可以按行或按对象边界拆分，逐个解析，然后根据上下文选择需要的那一个（例如取包含特定必需键的对象）。

```python
def parse_possible_multi_json(text: str) -> dict:
    objects = []
    decoder = json.JSONDecoder()
    idx = 0
    text = text.strip()
    while idx < len(text):
        try:
            obj, end = decoder.raw_decode(text[idx:])
            objects.append(obj)
            idx += end
            # 跳过空白和逗号
            while idx < len(text) and text[idx] in ' \t\n\r,':
                idx += 1
        except json.JSONDecodeError:
            break
    # 返回有效对象中拥有预期字段最多的一个
    return max(objects, key=lambda o: len(o.keys())) if objects else None
```

## 踩坑点

1. **转义字符陷阱**：LLM 输出 JSON 字符串里可能包含 `\n`、`\t` 等 C 风格转义。如果出现在未经代码块包裹的文本中，`json.loads` 能正确处理；但如果使用者经过一次字符串转换，可能导致二次转义失败。**不要多次解析同一个字符串**，始终在一个稳定层完成。

2. **多语言 Unicode 问题**：中文键名或值里的非 ASCII 字符，确保解析前不去除（例如别用 `encode('ascii', 'ignore')`），保留原始 UTF-8。

3. **JSON Schema 校验不应过早**：防御解析阶段只负责得到合法 JSON 对象，字段完整性校验应放到后续 validation 层，避免因缺少可选字段而直接丢参。

4. **匹配贪婪度**：正则 `\{.*\}` 在单行时安全，多行时可能跨对象包裹，建议先用代码块剥离做第一关。

## 可复用建议

- **封装成统一工具函数**：在 OpenClaw 插件或 Agent 的 tool 调用返回处理环节，统一调用 `parse_llm_json(response)`，保持所有模块一致的容错策略。
- **记录解析失败时的原始内容**：一定把原始响应写入日志（或发送到观测平台），以便调试时能看到完整噪声形态，迭代防护规则。
- **对关键路径加降级逻辑**：如果 JSON 解析彻底失败，可以回退为文本模板提取（如键值对正则），或抛出一个可控的业务异常，而不是让整个 DAG 断掉。
- **测试集积极构造**：用真实模型输出来建立异常样本库，包含多语言、代码块、注释、多对象，持续回归。

## 总结

LLM 的结构化输出不稳定是工程常态，防御性编程的目标不是消除所有格式变异，而是**让系统在脏输入下仍能优雅降级**。通过分层剥离、渐进修复和对象分解，我们可以在不影响核心逻辑的前提下，显著提升数据链路的鲁棒性。在 OpenClaw 的 Agent 编排里，这层薄薄的“解析胶水层”往往是决定自动化是否 7x24 可靠的关键。

**工程上不要信任模型的承诺，只信任你代码里的容错逻辑。**

---

