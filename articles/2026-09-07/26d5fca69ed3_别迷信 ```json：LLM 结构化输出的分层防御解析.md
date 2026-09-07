---
title: 别迷信 ```json：LLM 结构化输出的分层防御解析
feedId: 36434
source: 综合讨论
publishedAt: 2026-09-07
---

## 背景

在 OpenClaw 插件、MCP 工具和 Agent 流水线里，让模型输出 JSON 是最高频的需求：工具参数、节点间消息协议、自动化任务的中间产物。提示词里通常写死「用 ```json 代码块输出」，看起来很稳。但只要跑过真实负载就知道，格式漂移几乎必然发生。

## 问题

同一个 prompt，换模型、升温度、甚至同模型不同会话，输出可能是：

- 裸 JSON，没有围栏；
- 围栏标注写成 `JSON`、`jsonc` 或干脆不写；
- 围栏前后带一段解释文字；
- 尾逗号、全角引号、字符串里嵌套 ```；
- 把对象拆成两段代码块。

任何只按一种格式写的解析器，上线几天就会在生产里炸。

## 做法：分层降级

核心思路：不做一次性正则匹配，从成本最低的尝试开始逐层降级。

```python
import json, re

def parse_llm_json(text: str):
    # L0：直接解析（模型偶尔就是很乖）
    try:
        return json.loads(text)
    except json.JSONDecodeError:
        pass
    # L1：剥代码围栏，非贪婪 + 容忍语言标注
    m = re.search(r"```(?:json|jsonc|JSON)?\s*\n(.*?)```", text, re.S)
    if m:
        try:
            return json.loads(m.group(1).strip())
        except json.JSONDecodeError:
            text = m.group(1)
    # L2：括号配平截取（带字符串状态机），不是 first{ 到 last}
    start = text.find("{")
    if start != -1:
        depth, in_str, esc = 0, False, False
        for i, ch in enumerate(text[start:], start):
            if in_str:
                if esc: esc = False
                elif ch == "\\": esc = True
                elif ch == '"': in_str = False
            elif ch == '"': in_str = True
            elif ch == "{": depth += 1
            elif ch == "}":
                depth -= 1
                if depth == 0:
                    try:
                        return json.loads(text[start:i + 1])
                    except json.JSONDecodeError:
                        break
    # L3：修复类库兜底（json-repair / demjson 等）
    from json_repair import repair_json
    return json.loads(repair_json(text))
```

解析成功不等于结构正确。外层再套一层 pydantic 校验，失败就把具体报错回喂给模型重试，上限两次。重试 prompt 里明确写「你上次输出的问题在于 XXX」，比笼统地说「重新输出」成功率高得多。

## 踩坑点

1. **正则贪婪匹配**：`(.*)` 会把多个代码块并成一个；换成非贪婪 `(.*?)` 又会在字符串内含 ``` 时提前截断。所以围栏剥离只当 L1，不能当唯一手段。
2. **「第一个 `{` 到最后一个 `}`」**：正文里出现第二个独立 JSON、或示例代码含花括号时必炸。括号配平 + 字符串状态机是这类截取的下限写法。
3. **中文环境**：全角引号和冒号 `json.loads` 不认，直接落到 L3 修复层，别自己手写替换规则。
4. **流式输出**：别边收边 parse，先攒全再走分层；确实需要流式，用增量 JSON 解析器，并接受最后一层可能不完整。
5. **排障盲区**：解析失败时务必把原始文本脱敏后落日志，否则线上只能靠猜。

## 可复用建议

- 把 parse + validate + retry 封装成共享工具函数，所有插件统一调用，不要每份插件各写一套正则。
- 沉淀「坏样本语料库」：每次线上解析失败，脱敏后存为 fixture 加进单测。积累半年，你的解析器会比任何教程都结实。
- API 层支持结构化输出（response_format、tool calling 的 schema）就优先用，解析层只是给没有这层的模型和网关兜底。
- 调低温度对格式稳定有帮助，但替代不了防御性解析。

## 总结

LLM 的输出格式是概率性的，而解析器必须是确定性的——这个矛盾只能靠「分层降级 + 结构校验 + 失败回喂」来弥合。防御性解析不是不信任模型，而是让你的 pipeline 不必依赖这份信任。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/67153c622c486bc6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/cec812a550e5dedb.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/ae2e8ca2fe69107d.png)

