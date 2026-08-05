---
title: LLM 输出解析的防御性编程：JSON 标签格式混合处理
feedId: 31690
source: 综合讨论
publishedAt: 2026-08-05
---

## 背景

在 OpenClaw/Agent 开发中，我们经常要求 LLM 输出结构化数据，最常见的约定是 JSON。但实际运行中，模型返回的格式往往不是干净利落的纯 JSON——它可能被 Markdown 代码块包裹，可能夹在 XML 标签中间，也可能前后缀着解释性文字。如果你在集成 MCP 工具或自建插件时直接 `JSON.parse(response)`，迟早会被脏数据炸掉。

这不是模型能力问题，而是工程问题。既然格式漂移无法根除，解析端就必须具备防御性。

## 问题

我实际遇到的三种典型输出形态：

```text
// 形态一：Markdown 代码块包裹
```json
{"action": "read_file", "path": "/tmp/a.txt"}
```

// 形态二：XML/HTML 标签包裹
<output>
{"action": "search", "query": "OpenClaw 部署"}
</output>

// 形态三：前后有解释文字
好的，以下是解析结果：{"action": "list_dir", "path": "/home/user"}
```

如果你只处理第一种，第二种和第三种就会让你的 Agent 链路直接断裂。

## 做法与步骤

防御性解析不是玄学，核心就三件事：**剥离包裹层、提取 JSON 子串、容错解析**。以下是逐步实现：

**第一步：剥离 Markdown/XML 包裹**

按优先级顺序尝试提取：

```python
import re
import json

def extract_json_str(raw: str) -> str:
    # 1. 先找 ```json ... ``` 代码块
    fence_match = re.search(r"```(?:json)?\s*([\s\S]*?)```", raw, re.IGNORECASE)
    if fence_match:
        return fence_match.group(1).strip()
    
    # 2. 再找 <output>...</output> 或 <json>...</json> 标签
    tag_match = re.search(r"<(?:output|json|result)>\s*([\s\S]*?)</(?:output|json|result)>", raw, re.IGNORECASE)
    if tag_match:
        return tag_match.group(1).strip()
    
    return raw.strip()
```

**第二步：括号平衡提取**

如果模型没有任何包裹标记，只是把 JSON 混在文本里，就需要用花括号配对来截取：

```python
def extract_balanced_json(raw: str) -> str:
    start = raw.find('{')
    if start == -1:
        raise ValueError("No JSON object found")
    
    depth = 0
    in_string = False
    escape = False
    
    for i in range(start, len(raw)):
        ch = raw[i]
        if in_string:
            if escape:
                escape = False
            elif ch == '\\':
                escape = True
            elif ch == '"':
                in_string = False
        else:
            if ch == '"':
                in_string = True
            elif ch == '{':
                depth += 1
            elif ch == '}':
                depth -= 1
                if depth == 0:
                    return raw[start:i+1]
    raise ValueError("Unbalanced JSON brace")
```

注意：这段代码不能省略字符串状态判断。如果不判断 `in_string`，JSON 值中的花括号（例如 `{"content": "a{b}c"}`）会导致截断错误。

**第三步：容错解析**

如果标准 `json.loads` 失败，尝试 JSON5 风格容错：

```python
def robust_loads(text: str):
    try:
        return json.loads(text)
    except json.JSONDecodeError:
        # 去除尾逗号（LLM 常见错误）
        cleaned = re.sub(r",\s*([}\]])", r"\1", text)
        try:
            return json.loads(cleaned)
        except json.JSONDecodeError:
            # 单引号替换为双引号（仅限简单场景）
            alt = text.replace("'", '"')
            return json.loads(alt)
```

## 踩坑点

1. **顺序很重要**：必须先做代码块/标签剥离，再做括号平衡。反过来会截出包含 Markdown 标记的脏 JSON。
2. **字符串内部的花括号**：括号平衡必须识别字符串状态，避免在 `"{"` 内部计数。上面代码已处理，但很多自己写的版本会漏。
3. **截断问题**：如果 LLM 输出被 token 限制截断，JSON 不完整，`robust_loads` 也救不回来。这种场景下要判断错误是否包含 "Unbalanced" 或 "Expecting value"，返回重试信号而非静默降级。
4. **非 JSON 内容是合法的**：有时候模型确实没有输出 JSON（比如请求拒绝或工具不可用），这时要区分"解析失败"和"模型拒绝执行"，不要把两者混为一谈。

## 可复用建议

给正在做 MCP/插件集成的开发者几个建议：

- 把这套提取逻辑封装成独立函数，不要散落在业务代码里。所有需要 LLM 结构化输出的入口统一调用它。
- 建立回归测试样例：把历史遇到的脏格式（代码块嵌套、XML 标签错位、前后缀解释文字）全部加进测试集，每次改动先跑测试。
- 解析失败时，把原始响应记录到日志里。这些数据比任何 prompt 调优都值钱——它们是模型真实行为的样本。
- 如果 Agent 输出用于执行操作（比如操作文件、发请求），解析层二次校验 JSON schema 是必要的，单独做一步，不要合并进解析。

## 总结

防御性编程不是纵容模型乱输出，而是承认噪声不可避免。剥离包裹、括号平衡、容错解析三步走，能解决 95% 的格式混合问题。剩下 5%，靠日志和回归测试迭代。在 Agent 系统里，稳定性从来不是靠祈祷模型听话得来的——是用解析器兜底兜出来的。

---

