---
title: LLM 输出解析的防御性编程：打捞被 Markdown 和废话淹没的 JSON
feedId: 32529
source: 综合讨论
publishedAt: 2026-08-11
---

## 背景

在构建基于 LLM 的 Agent、MCP 工具链或自动化流水线时，几乎所有实践者最终都会撞上同一个问题：**模型的输出格式不稳定**。

我们通常需要从模型返回的文本中提取结构化数据——最常见的就是 JSON。理论上只需给出清晰的 system prompt 并要求“只输出 JSON，不要任何解释”，但现实是模型经常“创作欲过剩”：在 JSON 前后加上大段解释、把 JSON 包裹在 Markdown 代码块里、甚至输出多段“思考过程 + 最终 JSON”的混合体。更麻烦的是，即使模型真的只输出了一个花括号，也可能因为 token 截断、语法错误导致直接 `JSON.parse` 失败。

如果你的 Agent 或自动化流程无法健壮地处理这些“不干净”的输出，那 pipeline 的可靠性就会大幅下降。本文将整理一套**防御性解析策略**，处理 JSON + Markdown 标签 + 自由文本的混合输出，并给出可直接复用的工程建议。

---

## 核心问题

典型的“不干净”输出可能长这样：

```text
好的，我理解了你的问题。以下是分析结果：
```json
{
  "summary": "用户要求提取订单信息",
  "items": [{"sku": "ABC", "qty": 2}]
}
```
希望这对你有帮助。
```

或者更糟：

```text
思考：用户意图是查询物流状态，需要调用 search_order API...
{
  // 这是注释，但实际 JSON 不能有注释
  "status": "shipped",
  "tracking_url": "https://..."
}  // 此处可能还有多余字符
```

直接 `JSON.parse` 会失败。目标是从这样的噪声中**可靠地捞出合法的 JSON 对象**，并在实在无法解析时进行明确的回退。

---

## 防御性解析的做法与步骤

### 1. 剥离 Markdown 代码块

最常见的干扰是 ```` ```json ```` 和 ```` ``` ```` 标记。可以用正则将内容提取出来：

```python
import re

def strip_markdown_code(text: str) -> str:
    # 匹配 ```json ... ``` 或 ``` ... ```
    pattern = r'```(?:json)?\s*([\s\S]*?)```'
    match = re.search(pattern, text, re.DOTALL)
    if match:
        return match.group(1).strip()
    return text.strip()
```

注意要使用 `re.DOTALL` 让 `.` 匹配换行，否则跨行内容会丢失。

### 2. 从噪声中提取首个完整 JSON 对象

很多时候剥离代码块后仍然有前后文字。需要定位第一个 `{` 并使用栈匹配找到对应的 `}`，提取出 JSON 对象：

```python
def extract_json_object(text: str):
    start = text.find('{')
    if start == -1:
        return None
    stack = 0
    end = start
    for i in range(start, len(text)):
        ch = text[i]
        if ch == '{':
            stack += 1
        elif ch == '}':
            stack -= 1
            if stack == 0:
                end = i
                break
    if stack != 0:
        # 括号不匹配，可能被截断
        return None
    json_candidate = text[start:end+1]
    return json_candidate
```

这种栈式匹配可以正确处理 JSON 内部的嵌套对象，不依赖正则（正则处理嵌套花括号非常脆弱）。

### 3. 尝试解析并处理常见语法错误

拿到候选 JSON 字符串后，尝试解析。如果失败，可进行一些低风险的修复：

- **移除尾随逗号**（如 `"key": "value",}` 最后的逗号）。
- **将单引号替换为双引号**（但需要小心字符串内部的单引号，可以采用试探性解析，失败再替换）。
- **去除注释**：`// ...` 和 `/* ... */` 在标准 JSON 中不允许，可用正则移除。

在 Python 中可以这样尝试多级回退：

```python
import json

def robust_json_loads(s: str):
    # 先尝试直接解析
    try:
        return json.loads(s)
    except json.JSONDecodeError:
        pass

    # 移除行注释 (谨慎，避免 URL 中的 //)
    cleaned = re.sub(r'\/\/.*', '', s)
    try:
        return json.loads(cleaned)
    except json.JSONDecodeError:
        pass

    # 移除块注释
    cleaned2 = re.sub(r'\/\*[\s\S]*?\*\/', '', s)
    try:
        return json.loads(cleaned2)
    except json.JSONDecodeError:
        pass

    # 尝试修复尾逗号
    repaired = re.sub(r',\s*}', '}', s)
    repaired = re.sub(r',\s*]', ']', repaired)
    try:
        return json.loads(repaired)
    except json.JSONDecodeError:
        pass

    return None
```

### 4. 组合成统一的“打捞”函数

将上述步骤串起来：

```python
def safe_extract_json(text: str):
    # Step 1: 剥离 markdown 代码块
    stripped = strip_markdown_code(text)
    # Step 2: 提取首个 JSON 对象
    candidate = extract_json_object(stripped)
    if not candidate:
        return None
    # Step 3: 容错解析
    return robust_json_loads(candidate)
```

这个函数可以在 Agent 的解析节点或 MCP 工具的输出处理中直接调用。

---

## 踩坑点

1. **嵌套花括号计数错误**  
   栈匹配时若忽略字符串内的花括号（例如 `"url": "https://example.com/{id}"`）会导致提前退出。  
   解决：在遍历字符时判断是否处于字符串内。简单实现如下：

   ```python
   in_string = False
   escape = False
   for i, ch in enumerate(text[start:], start=start):
       if escape:
           escape = False
           continue
       if ch == '\\':
           escape = True
           continue
       if ch == '"':
           in_string = not in_string
           continue
       if in_string:
           continue
       # 处理花括号...
   ```

2. **JSON 数组与多对象**  
   模型可能返回一个 JSON 数组 `[...]`，上述提取逻辑需要调整：从 `[` 开始并匹配 `]`。同样地，也可能返回多个顶级 JSON 对象（如 `...}{...`）。大部分场景下只需第一个对象/数组，按需修改起点字符即可。

3. **注释移除误伤内容**  
   正则 `//.*` 会删除 URL 中的 `//` 部分。更稳健的做法是针对以 `//` 开头的独立注释行，替换时要求 `//` 前面只有空白符。简单场景直接移除即可，但当你解析的内容包含大量 URL 时需收窄条件。

4. **性能开销**  
   在多轮 Agent 循环中频繁运行正则和栈匹配可能带来轻微性能损耗。对于每步几十 KB 的输出，开销可忽略；但若处理巨量文本，可考虑先通过启发式（如是否以 `{` 开头）快速判断是否跳过清洗。

---

## 可复用建议

- **Prompts 前置防御**：在 system prompt 中加入“将 JSON 放在单个代码块内，不要包含任何其他内容”可以大幅减少混合输出。但永远不要完全信任模型。
- **将 `safe_extract_json` 模块化**：在每个需要解析模型输出的工具或插件中引入同一个解析模块，而不是到处手写正则。
- **灰度重试策略**：若解析失败，可以向同一模型发送一条简短的纠错提示（例如“你之前的输出不是有效 JSON，请纠正”）。这比重新生成全量回答成本更低。
- **日志记录**：将解析失败的原始输出写入日志，方便事后分析模型行为并持续优化提取逻辑。
- **单元测试**：收集你实际遇到的“脏”输出作为测试用例，确保修改解析逻辑后不会退化。

---

## 总结

LLM 输出是不可靠的渠道，混合 JSON、Markdown 标签和自然语言是常态而非异常。**防御性编程的核心不是消除不符合预期的输出，而是构建能容错的管道。** 通过剥离 Markdown 代码块、栈匹配完整 JSON、多级容错解析这么一套组合拳，可以大幅降低解析失败率，让基于 LLM 的自动化流程真正走向生产。

这不是银弹，但它是 Agent 和插件开发者工具箱里必须常备的一把钳子——不用多华丽，关键时刻能夹紧滑溜的输出就好。

---

