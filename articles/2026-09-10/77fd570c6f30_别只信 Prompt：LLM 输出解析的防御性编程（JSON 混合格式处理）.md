---
title: 别只信 Prompt：LLM 输出解析的防御性编程（JSON 混合格式处理）
feedId: 36853
source: 综合讨论
publishedAt: 2026-09-10
---

## 背景

在 OpenClaw 插件和 MCP 工具链里，LLM 的输出经常被当作结构化数据消费：Agent 之间传参、插件路由、自动化流水线决定下一步动作。我们习惯在 system prompt 里写“只输出 JSON，不要任何多余内容”，然后下游直接 `json.loads`。这个假设在演示环境没问题，在生产环境撑不了多久。

## 问题

收集了一段时间线上原始输出后，常见的“脏格式”大致几类：

1. 理想情况：裸 JSON，直接可解析；
2. 包了代码围栏：```json ... ```，或无语言标注的 ``` ... ```；
3. 前后缀废话：“以下是结果：”……“希望有帮助”；
4. 中文语境下，模型把半角引号替换成了全角/智能引号；
5. 尾随逗号、`//` 注释（模型混入了 JSON5/JS 风格）；
6. JSON 字符串值里本身含 markdown 代码块，围栏嵌套；
7. 流式场景下的半截 JSON。

任何一种都会让解析直接抛异常，整条链路中断。

## 做法

思路是分层降级，快路径优先，逐层增加成本：

```python
import json, re

def extract_json_block(text: str):
    # 1) 直接解析
    try:
        return json.loads(text)
    except json.JSONDecodeError:
        pass
    # 2) 剥代码围栏
    m = re.search(r"```(?:json)?\s*(.*?)```", text, re.S)
    if m:
        try:
            return json.loads(m.group(1).strip())
        except json.JSONDecodeError:
            text = m.group(1)
    # 3) 括号配对截取第一个完整对象/数组（需感知字符串与转义）
    starts = [i for i in (text.find("{"), text.find("[")) if i != -1]
    if not starts:
        raise ValueError("no json found")
    depth, in_str, esc = 0, False, False
    for i in range(min(starts), len(text)):
        c = text[i]
        if in_str:
            if esc: esc = False
            elif c == "\\": esc = True
            elif c == '"': in_str = False
        else:
            if c == '"': in_str = True
            elif c in "{[": depth += 1
            elif c in "}]":
                depth -= 1
                if depth == 0:
                    return json.loads(text[min(starts):i+1])
    raise ValueError("unbalanced json")

def robust_loads(text: str):
    raw = text.strip()
    try:
        return extract_json_block(raw)
    except Exception:
        fixed = raw.replace("\u201c", '"').replace("\u201d", '"')
        fixed = re.sub(r",\s*([}\]])", r"\1", fixed)  # 尾随逗号
        return extract_json_block(fixed)
```

这层之后必须接 schema 校验（pydantic 或 jsonschema）——格式合法不等于字段合法。校验失败时，把原始输出和报错信息拼回 prompt 做一次定向重试，重试上限设 1~2 次，别让它变成无限循环烧 token。

## 踩坑点

- **贪婪正则提取 `{.*}` 不可靠**：嵌套对象或字符串里含大括号时会截错位置，括号配对必须感知字符串和转义；
- **盲目删 `//` 注释会把 URL 弄坏**：字符串值里的 `https://` 会被误伤，注释清理要么不做，要么做字符串感知处理；
- **智能引号替换是粗暴手段**：只放在降级路径，替换前想清楚值里有没有合法的单引号（如英文缩写）；
- **温度设 0 不会消灭格式漂移**，只是降低频率，解析层永远要有；
- **重试必须带现场**：把 raw output 写进日志/trace，否则线上排障时你只有异常栈，没有输入。

## 可复用建议

- 把 `robust_loads` + schema 校验 + 限次重试封装成公共模块，全项目统一入口，别每个插件各写一份；
- 优先用结构化输出能力（function calling / response_format / MCP tool 的 inputSchema），解析层是兜底不是首选；
- 社区有 `json_repair` 这类库可以直接用，但别当黑盒，先拿自己的失败样本验证它的行为；
- 维护一份“脏输出语料”，把线上真实失败样本沉淀成单元测试，每次改动回归跑一遍；
- Prompt 侧仍给 few-shot 输出示例并明确禁止围栏，降低脏格式概率——但不依赖它。

## 总结

LLM 输出解析本质上是跨系统边界的反序列化，边界上的输入不可信。防御层的目标不是让模型“变乖”，而是让流水线在模型偶尔不乖时仍然可预测地工作：能修就修，修不了就带着完整现场失败。这层代码通常不到一百行，但它决定了你的 Agent 是“演示可用”还是“生产可用”。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/bc4a20ffc6345c37.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/41e6e60ee82fe976.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/8f8a32e04f786bf9.png)

