---
title: LLM 输出解析的防御性编程：JSON 与标签格式混合处理
feedId: 35202
source: 综合讨论
publishedAt: 2026-08-29
---

# LLM 输出解析的防御性编程：JSON 与标签格式混合处理

## 背景
在 OpenClaw 的 Agent 链路里，MCP 工具调用、插件回传、状态落盘通常约定 LLM 输出 JSON。但真实返回经常不是纯 JSON，而是代码围栏、自定义标签、解释文字和 JSON 混杂在一起。比如：

```text
好的，结果如下：
<result>
```json
{"tool":"search","args":{"query":"openclaw"}}
```
</result>
```

如果入口直接 `json.loads`，一个多余字符就会让整条自动化链路失败。防御性解析不是把提示词写得更凶，而是在代码层建立提取、容错、校验和回退的闭环。

## 问题定义
实际需要兼容的情况包括：

- ```` ```json ```` 代码围栏
- `<result>...</result>`、`<output>...</output>` 等自定义标签
- 标签和围栏混合嵌套
- JSON 前后夹杂自然语言说明
- 尾逗号、单引号、注释等非标准 JSON

处理目标是：从任意文本中稳定地提取 JSON 对象或数组，并转换成 schema 可校验的结构化数据。

## 做法 / 步骤

### 1. 建立提取流水线，不要依赖单点正则
按“剥围栏 -> 剥标签 -> 定位边界 -> 容错解析”的顺序处理：

```python
import json
import re
import json5  # pip install json5

def extract_json(text: str):
    original = text.strip()
    # 去掉零宽字符 / BOM
    original = original.encode("utf-8", "ignore").decode("utf-8-sig")

    # 1. 移除代码围栏
    fence = re.search(r"```(?:json)?\s*(.*?)```", original, re.S)
    if fence:
        original = fence.group(1).strip()

    # 2. 移除简单外层标签
    tag = re.search(r"<(\w+)>\s*(.*?)</\1>", original, re.S)
    if tag:
        original = tag.group(2).strip()

    # 3. 定位 JSON 边界
    start_candidates = [i for i in (original.find("{"), original.find("[")) if i != -1]
    if not start_candidates:
        raise ValueError("no json object/array found")
    start = min(start_candidates)
    end = max(original.rfind("}"), original.rfind("]"))
    candidate = original[start:end + 1]

    # 4. 容错解析
    try:
        return json.loads(candidate)
    except Exception:
        return json5.loads(candidate)
```

之后接 schema 校验，例如使用 Pydantic：

```python
from pydantic import BaseModel, ValidationError

class ToolCall(BaseModel):
    tool: str
    args: dict

try:
    data = extract_json(raw)
    call = ToolCall(**data)
except (ValueError, ValidationError) as e:
    # 进入修复/重试，并把 e 回传给模型
    ...
```

### 2. 失败时重试，而不是吞掉异常
解析或校验失败后，保留原始输出片段和错误信息，让模型根据报错重新生成。通常只需把校验错误（例如“args 不是 dict”）和原始输出一起发回，比泛泛重试更有效。

对 MCP 工具调用场景，解析成功后的 object 应直接作为标准输入传给工具，不要再传原始字符串，避免下游继续解析一遍。

## 踩坑点

**先剥围栏再剥标签。** 如果顺序反了，外层标签可能在代码围栏之外被误识别，导致剥错。上面的示例先处理围栏，再处理标签。

**边界定位要看上下文。** 用 `max(rfind("}"), rfind("]"))` 在解释文字里出现 `}` 时会误截。更稳的做法是先提取所有可能的对象/数组块，再用 schema 筛选；如果确认输出只有一个 JSON 主体，可以接受这个简化。

**正则不支持属性标签和嵌套。** `<result id="x">` 或 `<a><b>...</b></a>` 会让简单正则失效。如果自定义标签涉及属性，改成唯一 token 包裹，例如 `<<<result>>>`，或者先定位标签名再找对应结束标签。

**json.loads 不是终点。** `json5` 能处理尾逗号和注释，但对某些 `None`/`True` 大小写、NaN、未加引号 key 仍可能失败。可以再加 `json_repair` 作为第三级兜底。不要盲目尝试过多解析器，否则排障困难。

**流式输出不要边收边 parse。** 如果 OpenClaw 插件采用流式接收，先把片段聚合成完整候选块再解析。分片落在 JSON 字符串中间时，几乎所有边界判断都会失效。

## 可复用建议

- 把提取、修复、校验做成独立模块，所有 MCP/插件入口统一调用，不要在多个工具分支里写不同的小正则。
- 解析成功或失败都记录原始输出和走的解析路径，方便事后统计模型的“不配合率”。
- 优先用“剥离外层结构 + 定位 JSON 边界”，不要全局替换引号或字母，那会引入更难查的破坏。
- 设计自定义标签时，标签名要避开内容里可能出现的关键字；必要时用 `<<<`、`[[[]]` 这类少见边界。
- 严格重试提示词里提供“错误类型 + 原始片段 + 期望 schema”，不要只说“请输出合法 JSON”。

## 总结
在 OpenClaw/Agent/MCP 这类自动化链路里，LLM 输出格式的混合是常态，不是异常。防御性编程的意义在于：不依赖模型完全听话，而是在代码层建立提取、容错、校验、回退的闭环。一个小型解析器模块能显著降低插件和工具调用链路的脆弱性，同时保留日志和重试入口。它会让大多数“格式抽风”从致命错误降级为可重试的软异常。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/9464a51e6a30ccb7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/e5caecc4a10e9f7a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/4c965e1531b6ac37.png)

