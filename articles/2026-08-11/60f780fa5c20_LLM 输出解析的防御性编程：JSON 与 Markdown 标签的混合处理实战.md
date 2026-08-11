---
title: LLM 输出解析的防御性编程：JSON 与 Markdown 标签的混合处理实战
feedId: 32568
source: 综合讨论
publishedAt: 2026-08-11
---

# LLM 输出解析的防御性编程：JSON 与 Markdown 标签的混合处理实战

## 背景：当“结构输出”不再结构

在 OpenClaw、Agent、MCP 或任何依赖大语言模型输出结构化数据的自动化流程中，我们经常会这样要求模型：

> “请以 JSON 格式返回结果，格式为 { "action": "...", "params": {...} }”

然而，实际收到的原始输出往往是这样的：

```markdown
好的，这是分析结果：
```json
{
  "action": "search",
  "params": {"query": "OpenClaw agent 配置"}
}
```
希望这对你有帮助！
```

毫无疑问，直接 `json.loads()` 会抛出 `JSONDecodeError`。即便我们在 prompt 中千叮万嘱“只输出纯 JSON，不要任何额外文字”，模型的非确定性仍会导致：

- 头部或尾部出现解释性自然语言；
- 代码块包裹（ \`\`\`json ... \`\`\` 或 \`\`\` ... \`\`\`）；
- 多余的尾部逗号、注释等非标准 JSON 语法；
- 内容被截断或折叠。

这些问题是 LLM 对抗性输入的天然缺陷，无法通过 prompt 完全消除。**防御性解析**就成了必须建立的工程护栏——不信任模型输出的格式，而是用一套容错策略从中提取有效 JSON。

---

## 问题拆解：JSON 解析失败的常见面谱

在我们的自动化管线中，JSON 解析失败主要分为三类：

1. **外覆噪声**：输出含有额外文字或 Markdown 标记，有效 JSON 包裹在其中。
2. **语法瑕疵**：JSON 本身不严格符合规范，如尾随逗号、注释、键未加引号等。
3. **截断/多份输出**：模型输出了截断的 JSON，或在一次回答中返回了多个 JSON 对象，而我们只需要第一个有效数据。

仅靠简单的 strip 和单次 `json.loads` 显然不足以应对。

---

## 实践：构建一个鲁棒的 LLM JSON 提取器

下面给出一个逐步增强的防御性解析方案，采用 Python 实现，可无缝嵌入 OpenClaw 节点或任意 Agent 的解析层。

### 第一步：去除 Markdown 代码块

很多 LLM 会使用 ``` 包裹 JSON。先识别并提取包裹的主内容。

```python
import re

def extract_code_block(text: str) -> str:
    # 匹配 ```json ... ``` 或 ``` ... ```
    pattern = r"```(?:json)?\s*\n(.*?)\n```"
    matches = re.findall(pattern, text, re.DOTALL)
    if matches:
        return matches[0].strip()  # 优先采用第一个代码块
    return text  # 没有代码块则返回原文
```

**踩坑点**：当 JSON 字符串内部包含 `` ``` `` 时，`re.DOTALL` 可能会贪婪匹配错位。比如模型输出 JSON 中存在 Markdown 代码示例，这种情况极少，但一旦发生，可以回退为按行遍历，寻找首个 `{` 或 `[` 作为 JSON 起点。

### 第二步：提取首尾对齐的 JSON 候选串

从文本中定位第一个 `{` 或 `[` 到最后一个 `}` 或 `]`，排除外围噪声。

```python
def extract_json_substring(text: str) -> str:
    # 寻找第一个 { 或 [
    start_idx = -1
    for i, ch in enumerate(text):
        if ch in ('{', '['):
            start_idx = i
            break
    if start_idx == -1:
        return text  # 没有 JSON 结构，交由后续处理抛异常
    # 寻找最后一个 } 或 ]
    end_idx = -1
    for i in range(len(text)-1, -1, -1):
        if text[i] in ('}', ']'):
            end_idx = i
            break
    if end_idx == -1 or end_idx < start_idx:
        return text
    return text[start_idx:end_idx+1]
```

此方法简单有效，但对截断 JSON 无效（比如最后一个对象不完整）。此时可结合异常容错库。

### 第三步：结合 json_repair 处理语法错误

对于尾部逗号、注释、键未加引号等非标准 JSON，[json_repair](https://github.com/mangiucugna/json_repair/) 这类专门修复 JSON 的库比手写正则可靠得多。

```python
from json_repair import repair_json
import json

def parse_llm_json(text: str) -> dict:
    # 预处理
    cleaned = extract_code_block(text)
    candidate = extract_json_substring(cleaned)
    # 尝试直接解析
    try:
        return json.loads(candidate)
    except json.JSONDecodeError:
        pass
    # 用 repair 修复后再解析
    try:
        repaired = repair_json(candidate)
        return json.loads(repaired)
    except Exception:
        # 最终兜底：抛出描述清晰的异常
        raise ValueError(f"无法从 LLM 输出中解析 JSON: {text[:200]}")
```

**踩坑点**：

- `repair_json` 在输入包含非 JSON 外围文字时会失败，所以必须先提取 JSON 候选串。
- 对于嵌套对象截断（如 `{"a":{"b":"c`），修复库可能补全为错误结构。此时应根据业务设定超时重试机制，而不是死解析。
- 修复库会容忍一些离谱语法，可能产生非预期的数据结构，需在解析后增加 schema 验证。

### 第四步：结合 prompt 降级策略

防御性代码之外，prompt 设计也能降低解析难度：

- 明确指定输出只含 JSON，并要求开头就是 `{` 或 `[`。
- 提供 `stop` token（如 `}` 后的自定义结束符），减少解释性后缀。
- 使用 function calling 特性的模型时，优先走官方 tool/function 模式输出结构化对象，可省去解析。

但即使这样，防御解析仍是底线。

---

## 踩坑点总结

1. **代码块内存在多个 JSON**：如模型返回“方案一 JSON … 方案二 JSON …”。提取规则应定为“第一个可解析的对象”。
2. **JSON 内嵌 Markdown**：极罕见但可能导致代码块提取出错，需回退到括号定位。
3. **大 JSON 导致超时**：在 OpenClaw 的流式消费中，可能收到不完整的 JSON，解析逻辑最好与流结束校验结合。
4. **编码问题**：极少数模型会在 JSON 字符串中加入特殊 Unicode 或转义不完整，`repair_json` 可能失效，可尝试 `ast.literal_eval` 作为最后手段（仅限安全环境）。

---

## 可复用建议

封装一个通用工具 `parse_llm_json`，放入项目 common 模块，并暴露以下几个控制参数：

- `strict`：是否强制 schema 检查（配合 pydantic 或 JSON Schema）。
- `multiple`：是否尝试解析多个 JSON（返回列表）。
- `fallback`：当所有解析失败时，是否返回 None 而非抛异常，以便流程优雅降级。

将其与 OpenClaw 的节点输出挂钩，例如：

```python
output = call_llm(prompt)
try:
    data = parse_llm_json(output, strict=True)
except ValueError:
    log.error(f"非法 JSON 输出: {output[:200]}")
    # 触发重试或人工审核
```

这样所有依赖 LLM 结构输出的节点都获得一致的鲁棒性。

---

## 总结

在 LLM 驱动的自动化中，**解析输出永远要假设最坏情况**。混合 JSON 与 Markdown 标签的输出是常态而非例外。通过“代码块提取 → 候选串定位 → 容错修复 → schema 验证”的级联防御，我们可以把解析失败率降低两个数量级。配合 prompt 降级和输出监控，能构建出生产环境下足够健壮的 Agent 管线。

防御性编程不是对模型的不信任，而是对工程可靠性的尊重。

---

