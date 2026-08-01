---
title: LLM 输出解析的防御性编程：当 JSON 混进标签时怎么稳稳接住
feedId: 31275
source: 综合讨论
publishedAt: 2026-08-02
---

## 背景：为什么看上去简单的提取实际一跑就翻车

在 OpenClaw、MCP 插件、或者任何“LLM 作为子程序”的自动化场景里，我们都免不了要让模型返回结构化数据。最常见的要求就是：“请只输出一个 JSON 对象”。但真实工程环境里，模型输出的并不是一个干净的 `{"key": "value"}`，而更像是下面这种混合体：

```text
Sure, here's the result you asked for:
```json
{
  "status": "ok",
  "data": [
    { "id": 1, "name": "Alice" },
    { "id": 2, "name": "Bob" }
  ]
}
```
Let me know if you need anything else.
```

如果你直接用 `JSON.parse()` 去套，一百次里能成功七十次就不错了。剩下的三十次可能是：
- 前后有说明文字；
- 被 Markdown 代码块包裹，但漏了语言标识或者用了 ` ```json ` 以外的标记如 ` ``` `；
- JSON 里有尾随逗号或注释；
- 输出被意外截断，最外层括号不匹配；
- 同时要求模型输出 `<thinking>` 和 `<result>` 标签，但模型把 JSON 塞进了不规则的标签里。

在 Agent/MCP 链路里，这种不可靠性会被放大：一个环节的解析失败有可能直接中断整个工作流。所以我们需要为“输出解析”单独写一套防御性逻辑，让它能扛住模型的各种不稳定行为。

## 问题拆解：我们在解析时究竟要应付什么

归纳下来，LLM 输出的结构化数据主要有三种形式的变异：

1. **包裹形式变异**：纯 JSON、``````json``````、``````````、`<output>...</output>`、`<json>...</json>` 等。
2. **内容位置变异**：JSON 可能不在字符串开头，被文字包围。
3. **语法细节变异**：单引号、尾随逗号、注释（`//` 或 `/* */`）、数字前缀加号、NaN/Infinity 等非标准 JSON 值。

我们必须能识别并稳定地从中提取出“那份真正的 JSON”，再把它修正到可被解析的状态。

## 实战做法：一套尽量通用的解析器

下面的方案不依赖大而全的库，核心逻辑只用到标准库和 `json5`（对非标准 JSON 容忍度更高）。过程分三步：**定位候选段 → 提取 JSON 字符串 → 宽松解析与报错恢复**。

### 1. 定位候选段：标签与代码块优先，再 fallback 到首个 `{` 或 `[`

先用正则取出所有代码块内容和标签包裹内容，如果恰好只有一个，就当作 JSON 候选。如果存在多个，选第一个非空的。都没有匹配到，就从输出里找到第一个 `{` 或 `[`，并尝试找对应的闭合符号。

```python
import re

def extract_json_blocks(text: str):
    # 提取 fenced code blocks
    fenced = re.findall(r'```(?:\w+)?\s*\n?(.*?)\n?```', text, re.DOTALL)
    # 提取 <output>...</output> 或 <result>...</result>
    tag_blocks = []
    for tag in ['output', 'result', 'json']:
        matches = re.findall(f'<{tag}>(.*?)</{tag}>', text, re.DOTALL)
        tag_blocks.extend(matches)
    return fenced + tag_blocks
```

如果上面拿不到内容，再进入括号定位：使用一个简单的平衡括号查找器，从第一个 `{` 或 `[` 开始，推入栈直到栈空或结束，取到的子串就是可能的 JSON 块。

### 2. 清理与修复：去掉外层包裹，修正已知的语法问题

拿到候选字符串后，去掉首尾空白。如果还被引号包裹（极少数情况模型可能输出 `"{...}"`），剥掉一层引号。

接着交给 `json5` 做宽松解析。`json5` 可以容错处理：
- JSON5 标准本身的尾随逗号、单引号、十六进制数字等；
- 还能处理一些额外的非标准情形，比如注释。

```python
import json5

def safe_parse(candidate: str):
    try:
        return json5.loads(candidate)
    except Exception as e:
        # 记录日志，但不在生产里直接抛出原始异常
        raise ValueError(f"JSON5 parse failed: {e}")
```

### 3. 失败兜底：重试、裁剪或让模型自己修

如果以上步骤都失败，在 Agent 流程里可以触发一次重试，把错误信息（不要暴露原始输出）传给 LLM，让它重新生成。例如：

> Your last response could not be parsed as JSON. Please return ONLY a valid JSON object, without any markdown fences or surrounding text.

注意，重试时**必须提示模型只输出纯 JSON**，且建议加限制词如 “Do not wrap in code blocks.”。

## 踩坑点集合（亲测有效）

1. **大括号在字符串里捣乱**  
   如果 JSON 内部某个字段值包含 `}`，简单的正则 `{.*}` 会提前截断。必须用带括号计数的提取方式，不能用贪婪或非贪婪正则直接取大括号对。推荐自己写个简单的平衡计数器，跳过字符串字面里的括号。

2. **模型有时会输出 Python 的 `None` 而不是 `null`**  
   `json5` 默认不处理 `None`、`True`、`False`。在解析前可以用简单替换修正：
   ```python
   candidate = re.sub(r'\bNone\b', 'null', candidate)
   candidate = re.sub(r'\bTrue\b', 'true', candidate)
   candidate = re.sub(r'\bFalse\b', 'false', candidate)
   ```

3. **截断输出**  
   有时模型输出 4096 后截断，JSON 不完整。这时平衡括号查找会到末尾栈不为空。可以尝试用 `json5` 的宽松模式（有些实现可以忽略末尾问题），但更稳健的做法是直接判定失败并重试，或使用更短的输出要求。

4. **多 JSON 对象混在一条消息里**  
   如果你让模型一次性输出多个 JSON 对象（比如一行一个），但它们没有外层数组包裹。可以按行或按分割符拆开逐个解析，而不是试图用同一个提取逻辑。

## 可复用的工程建议

- **封装为独立函数**：输入原始字符串，输出 dict/list，或抛出明确的自定义异常（如 `OutputParseError`）。
- **支持配置解析策略**：如优先使用代码块/标签，还是直接从括号起始位置分析。
- **保留原始输出日志**：万一解析失败，完整的原始输出对调试至关重要，但不要原封不动返回给用户。
- **与重试机制解耦**：解析器只负责解析，重试逻辑交给上游的 Orchestrator，方便单独测试和复用。
- **单元测试用真实模型输出样例**：准备至少 20 条不同格式的歪输出，覆盖上面所有变异情况，保证迭代时不回退。

## 总结

在 LLM 应用的自动化链路里，输出解析的健壮程度通常决定了系统是“看着挺智能”还是“真能跑得动”。JSON 和标签混合的防御性解析不是银弹，但通过“候选块提取 → 语法宽松化 → 失败兜底”的三段式设计，可以把不可控的概率压缩到一个可以接受的角落。最终目标不是百分之百解析成功，而是当失败发生时，你有清晰的路径让它重新回到正轨。

---

