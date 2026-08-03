---
title: 防御性解析实战：当 LLM 输出同时混进 JSON 和自定义标签
feedId: 31442
source: 综合讨论
publishedAt: 2026-08-03
---

## 背景

在基于 LLM 的 Agent、MCP 工具调用或插件自动化链路中，我们通常会要求模型返回结构化 JSON，以便下游直接反序列化。但真实环境中，即便用了 system prompt 强调“只输出 JSON”，模型依然经常“多嘴”——在 JSON 前后塞入解释性文本、用 Markdown 代码块包裹，甚至使用类似于 `<json>...</json>`、`<response>...</response>` 的自定义标签。这类混合输出会让脆弱的 `json.loads()` 直接爆炸，整条自动化链路随之断掉。

OpeClaw 生态不少插件和自动化流程都依赖 LLM 作为数据提取或决策节点，如果不能宽容地处理这些“小意外”，系统根本称不上稳健。本文记录一套在实践中验证过的防御性解析策略，专门应对“JSON + 标签 + 文本” 混合体。

## 典型问题输出长什么样

实际项目中反复出现的几种形态：

1. **Markdown 代码块包围**
````
Here is the result:
```json
{"action": "search", "query": "how to parse JSON"}
```
Let me know if you need anything else.
````

2. **自定义 XML/标签包裹**
````
<json>
{
  "tool": "read_file",
  "path": "/tmp/demo.py"
}
</json>
````

3. **多块混杂（只有其中一块是目标数据）**
````
Here are the options:
```json
{"mode": "list"}
```
```json
{"mode": "detail", "id": 42}
```
Choose the one that fits.
````

4. **标签内带“修饰”**
````
<tool_call>
{"name": "send_message", "args": {"text": "hello"}}
</tool_call>
````

甚至还会出现 BOM 头、零宽空格、中文标点、尾部多余逗号。任何单一路径的解析器一遇到这些变体就容易失效。

## 防御性解析的做法与步骤

核心思路是建立一个**优先级的回退链**，先用最严格的方式解析，失败后逐步放宽假设。大致流程如下：

### 1. 原始输出消毒
拿到 `raw_output: str` 后，先做轻量清洗：
- 移除 BOM、零宽字符（`\u200b`、`\u200c` 等）
- 统一换行符为 `\n`
- 可选：将中文引号、全角符号置换为半角（当确定 JSON 内容不应含全角时）

这一步能消灭大量因编码或平台复制导致的隐形字符问题。

### 2. 优先尝试直接解析
对清洗后的文本直接做 `json.loads()`，并捕获异常。如果成功，直接返回，这是最快路径。

### 3. 提取 Markdown 代码块
若直接解析失败，用非贪婪匹配或逐行扫描提取 ```` ```json ```` 和 ```` ``` ```` 之间的内容。注意处理：
- 缺乏语言标识的普通 ` ``` ` 代码块
- 同一输出中有多个代码块时，需约定优先顺序（例如取最后一个，或取包含特定 key 的块）
- 代码块内部可能还有嵌套反引号，一般不需要处理，只要保证闭合即可

提取到候选字符串后，回到步骤 2 尝试解析。

### 4. 识别并提取自定义标签
这是“标签格式混合处理”的关键。常见标签包括：
- `<json>...</json>`
- `<response>...</response>`
- `<tool_call>...</tool_call>`
- `<function_cast>...</function_cast>`

实现上，可以用正则 `re.search(r'<\w+>(.*?)</\w+>', text, re.DOTALL)` 作为宽松匹配，但为了避免误匹配（例如文本中出现一对 HTML 标签），最好维护一个已知标签名列表，按优先级尝试。提取到文本后同样回到步骤 2。

**踩坑点**：大小写不敏感、标签内可能存在空格（`<json >`）、模型因为 token 边界把标签截断等。实践中至少要对标签名做 `trim` 和大小写无视处理。

### 5. 最外层花括号扫描
如果以上都失败，退化为正则寻找第一个 `{` 和最后一个 `}`，并手工提取平衡部分。简单实现可以用计数器变量从第一个 `{` 开始累加深度，直到深度归零。这种方法能处理大部分“JSON 前后带闲聊”的场景，但若正文中夹杂其他接近 JSON 的大括号文本，容易误识别。因此应将其作为最后一步，并设置提取结果的最小长度阈值（例如大于 6 个字符）。

### 6. 使用宽松解析器兜底
提取出的字符串可能包含 JavaScript 风格的注释、尾随逗号等非标准 JSON 语法。此时可以引入 `json5` 库（或类似实现）进行解析。如果环境不允许引入额外依赖，可对提取出的字符串做一次正则清洗：移除 `//` 和 `/* */` 注释，去掉尾随逗号，再交给标准库。

### 7. 全流程简化伪代码

```python
import re, json
from typing import Optional

def defensive_json_parse(raw: str) -> Optional[dict]:
    text = sanitize(raw)

    # 1. direct
    result = try_loads(text)
    if result is not None:
        return result

    # 2. markdown fenced blocks
    blocks = re.findall(r'```(?:json)?\s*\n(.*?)\n```', text, re.DOTALL)
    for block in reversed(blocks):  # 取最后一块
        result = try_loads(block)
        if result is not None:
            return result

    # 3. known tags
    for tag in ('json', 'tool_call', 'response'):
        m = re.search(rf'<{tag}\s*>(.*?)</{tag}>', text, re.DOTALL|re.IGNORECASE)
        if m:
            result = try_loads(m.group(1))
            if result is not None:
                return result

    # 4. greedy brace extraction
    start = text.find('{')
    if start != -1:
        brace_level = 0
        for i in range(start, len(text)):
            if text[i] == '{':
                brace_level += 1
            elif text[i] == '}':
                brace_level -= 1
                if brace_level == 0:
                    candidate = text[start:i+1]
                    result = try_loads(candidate)
                    if result is not None:
                        return result
                    break
    return None

def try_loads(s: str) -> Optional[dict]:
    try:
        return json.loads(s)
    except json.JSONDecodeError:
        try:
            return json5.loads(s)   # 需要有 json5 库，无则注释
        except:
            return None
```

## 实际踩坑点补充

- **贪婪匹配陷阱**：在使用 `<.*>.*</.*>` 这类正则提取标签时，如果输出中存在多个相同标签，`re.DOTALL` 可能把第一个 `<json>` 和最后一个 `</json>` 之间的所有内容都当成一个块。一定要使用非贪婪 `.*?`。
- **Markdown 代码块嵌套**：极少见但存在。例如模型输出里包含一段代码块用于展示如何输出 JSON，然后外层还有代码块。此时只提取最内层往往也没问题，但需要关注嵌套深度。可改为按 ``` 切分后分析。
- **模型返回不完整 JSON**：流式输出长 JSON 时可能被 token 限制截断，防御性解析无法解决截断问题，但可以在提取时配合记录日志和告警，提醒需重试。
- **数字类型强制**：如果期望值是固定 schema，解析后应再做一层校验，而非仅检查 `dict` 类型。

## 可复用建议

1. **封装成标准工具函数**：团队内所有解析 LLM 结构输出的地方都调用同一个 `parse_llm_json`，避免各处散落脆弱实现。
2. **始终保留原始输出**：在日志中记录 `raw_output`，方便日后复盘新增的怪异输出模式，并根据反馈补充提取规则。
3. **配置化标签列表**：将允许的自定义标签名作为配置项，方便不同 Agent 定制（如 `tool_call`、`action` 等）。
4. **失败时不要静默**：如果最终返回 `None`，应当抛出业务异常或触发重试，严禁随手给个空对象继续运转，那会把错误向后推。
5. **性能考量**：正则和循环扫描本身开销很小，不必过多担心；但在高并发场景，避免对超长文本频繁回溯。

## 总结

把 LLM 的输出当作“天然不可信”是构建稳健 Agent 流水线的前提。防御性解析不是简单地砍掉 ` ``` `，而是设计一条从严格到宽松、可扩展的回退路径。JSON 与标签共存的场景，在现实项目中很难彻底避免，与其花更多精力去优化 prompt 试图杜绝，不如提前在解析器上建立安全边际。

这一策略已经在多轮对话驱动的工具调用、MCP 数据提取的自动化流程中稳定运行，希望能为 OpeClaw 社区里同样受困于“不听话输出”的同学提供一点思路。

---

