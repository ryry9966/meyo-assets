---
title: LLM 输出解析的防御性编程：JSON 与标签格式混合的鲁棒处理
feedId: 31772
source: 综合讨论
publishedAt: 2026-08-05
---

在 OpenClaw 生态中，无论是自定义 Agent 的工具调用、MCP 插件的数据交换，还是自动化流水线里对 LLM 输出的下游消费，我们都习惯通过 Prompt 要求模型返回结构化 JSON。但实际拿到的字符串往往离“纯净 JSON”很远：有的被包裹在 ```json代码块中，有的嵌在 `<output>` 等自定义标签内，有的前面带着“Here is the result:”之类的自由文本，甚至还有单引号、尾部逗号、注释等非标 JSON 痕迹。

如果把 `json.loads()` 直接怼上去，运行一段时间后大概率会被奇怪的输出击穿。这就要求我们在解析层引入防御性编程——不对输入做任何完美假设，用多级回退兜住各种异常格式，同时完整保留原始输出以供排查。

## 一、分层防御策略

以下是一个在 Agent 工具中实测有效的解析流水线，每一步都是一个可独立的 fallback。

### 1. 原始输出预处理
先 `text.strip()` 去除首尾空白。不要贸然截断，因为有时 JSON 前面确实会有引导语，但没必要用复杂 NLP 去识别，后面的步骤会自然处理。

### 2. 代码块提取
大量模型习惯用 Markdown 代码块包裹 JSON：
```python
import re

def extract_code_block(text: str) -> str:
    pattern = r"```(?:json)?\s*([\s\S]*?)\s*```"
    match = re.search(pattern, text)
    return match.group(1) if match else text
```
匹配到后取第一组内容，继续后续处理；如果未命中则原样传递。这里使用非贪婪并打开 re.DOTALL 等效模式（[\s\S]），防止换行截断。如果输出包含多个代码块，提取第一个通常就是有效 JSON。

### 3. 标签包裹解析
对于像 `<function_call>{"name": "..."}</function_call>` 这样的混合格式，可以先用正向/反向捕获提取标签内容：
```python
def extract_tag(text: str, tag: str = "output") -> str | None:
    pattern = rf"<{tag}>\s*([\s\S]*?)\s*</{tag}>"
    m = re.search(pattern, text)
    return m.group(1) if m else None
```
提取后再递归交给 JSON 解析器。如果同时出现多个不同标签，可根据业务优先级依次尝试。

### 4. JSON 解析容错
拿到可能为 JSON 的字符串后，不要直接 `json.loads()`，而是按以下顺序尝试：

- **标准 json.loads**：如果能一次成功，说明格式干净，直接返回。
- **使用 json5 库**：`pip install json5`，它能容忍单引号、尾部逗号、`//` 注释等常见畸变，对 LLM 输出特别友好。
- **轻量修复函数**：如果不想引入额外依赖，可以写一个简单的修复器，在 `json.loads` 前做：
  - 将独立出现的单引号替换为双引号（小心字符串内部）
  - 移除末尾逗号（通过简单正则 `,(\s*[}\]])` 替换为 `\1`）
  - 将 `True`/`False`/`None` 转换为小写（如果模型用了 Python 风格）
  但这种方式健壮性有限，生产环境建议直接拥抱 json5。

### 5. 正则兜底与“捞 JSON”
如果上述全都失败，说明字符串里可能夹带了大量自由文本，但其中仍藏着一个 JSON 对象或数组。此时可以写一个最外层提取函数，用字符平衡方法找到第一个完整 `{...}` 或 `[...]`：
```python
def brute_extract_json(text: str) -> str | None:
    # 找第一个 '{' 或 '['，然后计数括号平衡
    start_idx = min((text.find(c) for c in '{[}' if text.find(c) != -1), default=-1)
    if start_idx == -1:
        return None
    stack = []
    for i, ch in enumerate(text[start_idx:], start_idx):
        if ch in '{[':
            stack.append(ch)
        elif ch in '}]':
            if not stack:
                return None
            if (ch == '}' and stack[-1] == '{') or (ch == ']' and stack[-1] == '['):
                stack.pop()
            else:
                return None
        if not stack:
            return text[start_idx:i+1]
    return None
```
拿到子串后再用 json5 解析。这个策略能救回大量“脏输出”。

### 6. 多对象与多行场景
如果模型在流式或逐行场景下返回多个 JSON 对象（如每行一个 JSON），可以按行分割后逐行尝试解析，收集成功结果。并在架构上要求下游能处理列表。

### 7. 校验与日志
解析成功后务必用 Pydantic 或 JSON Schema 做字段校验，不合法也视为失败。**所有失败分支必须记录原始文本**和错误类型，否则线上问题无法追溯。在生产 Agent 里，这些日志比“准确率”更重要。

## 二、踩坑实录

- **中文引号**：模型偶尔输出 `“key”` 这种全角引号，需要在预处理阶段统一替换。
- **标签内换行**：标签提取必须用 `re.DOTALL` 等价写法，否则换行导致匹配中断。
- **Thinking 块干扰**：某些 Agent 框架会把 `<thinking>` 剥离，导致后续 JSON 残缺。需在解析链路中尽早处理标签，或者先整体解析再剥离 thinking。
- **json5 并不万能**：如果 JSON 里嵌有 Markdown 代码块，json5 仍会失败。这时候只能靠正则捞取加人工确认。
- **性能**：多层 try-except + 正则匹配在高并发下会有微小开销，但 Agent 调用通常频率可控，稳定性收益远大于这点消耗。

## 三、可复用工程建议

把这些逻辑封装成一个 `OutputParser` 类，提供类似 `parse_json(text, tag=None)` 的接口。内部使用**责任链模式**：一系列 Parser 依次尝试，第一个成功即返回。在 OpenClaw 的自定义工具装饰器或 MCP 适配层中植入，可以显著降低每个插件重复实现解析逻辑的负担。

另外，为每个 Agent 构建一套回归测试用例，把历史上出现过的畸形输出全部固化为 test case，防止后续修改破坏兼容性。

---

**总结**：LLM 输出从不可靠走向可靠，靠的不是更强的 Prompt，而是一层层防御代码。接受概率模型的混乱，用多级回退、json5 容错、正则兜底和完整日志，把解析做“软”了，整体链路反而更“硬”。这种工程化思维，是每个 OpenClaw/Agent 开发者必须内建的基础件。

---

