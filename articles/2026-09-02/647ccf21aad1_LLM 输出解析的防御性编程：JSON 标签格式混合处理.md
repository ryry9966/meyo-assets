---
title: LLM 输出解析的防御性编程：JSON 标签格式混合处理
feedId: 35759
source: 综合讨论
publishedAt: 2026-09-02
---

## 背景

在 OpenClaw 插件和自动化链路里，让 LLM 返回 JSON 是最常见的结构化交互。但只要跑过真实批量的任务就会发现：同一个 prompt，不同模型、不同温度、甚至不同会话，返回的"JSON"形态五花八门——有的包在 ```json 围栏里，有的是裸 JSON，有的前后带一句"以下是结果："，还有的带尾逗号、单引号，甚至混着 `<think>` 推理段。解析层如果不做防御，跑批必然三天两头挂。

## 问题

最典型的翻车写法是 `json.loads(response)` 一把梭。模型只要多输出一个围栏或多说一句话，插件就抛异常退出。更隐蔽的坑在流式场景：每帧拿到的都是半截 JSON；以及模型输出单引号对象、`True/None` 这类 Python 风格字面量时，标准 `json` 模块直接拒收。

## 做法

思路是把解析当成一条**分层降级的流水线**，而不是一次调用：

```python
import json, re
from json_repair import repair_json

def extract_json(text: str):
    # 1) 预清洗：去 think 标签、代码围栏
    text = re.sub(r"<think>.*?</think>", "", text, flags=re.S).strip()
    fence = re.match(r"^```(?:json)?\s*(.*?)\s*```$", text, flags=re.S)
    if fence:
        text = fence.group(1).strip()
    try:
        return json.loads(text)
    except json.JSONDecodeError:
        pass

    # 2) 括号配对截取（跳过字符串内部的括号）
    for start in (i for i, c in enumerate(text) if c in "{["):
        depth, in_str, esc = 0, False, False
        for i in range(start, len(text)):
            c = text[i]
            if esc: esc = False; continue
            if c == "\\": esc = True; continue
            if c == '"': in_str = not in_str; continue
            if in_str: continue
            if c in "{[": depth += 1
            elif c in "}]}":
                depth -= 1
                if depth == 0:
                    try:
                        return json.loads(text[start:i + 1])
                    except json.JSONDecodeError:
                        break
    # 3) 兜底修复
    return json.loads(repair_json(text))
```

解析成功不等于能用。第四步用 pydantic 校验字段名和类型，第五步才是反溃重试：把**解析错误信息 + 上一次的原始输出片段**一起回喂给模型，明确要求"只输出 JSON，不要围栏"，同时把温度降到 0~0.2。实践中两轮重试能解决九成以上的格式漂移。

## 踩坑点

- **括号配对忽略字符串内的 `{`**：值里有 `"prefix{xxx"` 就会切错位置，必须维护 `in_str` 状态。
- **正则贪婪匹配**：用 `\[.*\]` 提取数组，嵌套时会截到中间某个闭合符。
- **剥围栏误伤内容**：JSON 字符串值里可能出现反引号，判断围栏要用 `startswith`/正则锚定首尾，不要盲目 `replace`。
- **修单引号误伤撇号**：全局把 `'` 换成 `"` 会破坏 `it's` 这类值，交给 json_repair 这类库处理更稳。
- **重试不带上下文**：只说"格式错了"，模型很可能原样再犯。

## 可复用建议

- 把解析器抽成独立模块，所有插件共用，并记录每次"做了什么修复动作"，积累回归样本。
- prompt 层做双保险：给一个 few-shot 输出示例 + 显式声明"只输出 JSON"。
- 把**解析失败**和**schema 校验失败**分开计数，前者是格式问题，后者往往是模型能力或 prompt 表述问题，治理手段不同。
- 关键链路优先用 function calling / structured output，自由文本 + 正则只做兜底。

## 总结

LLM 的输出格式天然不可靠，解析层的目标不是"祈祷它对"，而是**假设它一定会以各种姿势出错**，然后用清洗、定位、修复、校验、重试五层防线兜住。这套东西一次写好，所有插件受益。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/c96a407ef139abf7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/7c87c53eebef18b1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/f5d3cc461a80ab8d.png)

