---
title: LLM 输出解析的防御性编程：JSON 标签格式混合处理
feedId: 34987
source: 综合讨论
publishedAt: 2026-08-28
---

# LLM 输出解析的防御性编程：JSON 标签格式混合处理

## 背景

在 OpenClaw、Agent、MCP、插件自动化链路里，我们经常默认模型会“乖乖地”返回 JSON，或至少返回一个干净的 function call。现实是：一旦接入第三方模型、长上下文任务、低温度采样，或者工具约束不够强，输出就会变成混合格式：

```text
Here is the result:

```json
{
  "action": "click",
  "x": 10,
  "y": 20,
}
```

Please confirm.
```

还可能更糟：前导解释文字、两个 JSON 块、`{'x': 10}` 这种 Python dict 风格、`None` 混在 JSON 里、代码块大小写不统一、甚至示例 JSON 和真实 JSON 同时出现。

如果解析层只写一句 `json.loads(text)`，线上很快就会收到一堆 `Expecting value: line 1 column 1` 或 `Expecting property name enclosed in double quotes`。对自动化流程来说，这不是偶发 bug，而是必然要处理的一类输入。

## 问题

LLM 输出不是可靠协议，而是“自然语言 + 结构化尝试”的混合体。防御性编程的核心是：不信任输出格式，把解析过程变成候选提取、低风险修复、schema 筛选、失败回传四步。这样既能兼容大多数混合格式，又能保留原始输出便于排障。

## 做法/步骤

我建议把解析逻辑拆成两个函数：`extract_json_candidates(text)` 和 `parse_llm_payload(text, schema=None)`。

第一步是候选提取。优先提取 fenced code block，再扫描裸 JSON 块。示例实现如下：

~~~python
import re

def extract_json_candidates(text: str):
    candidates = []
    # 1. 提取 ```json / ```JSON / ``` 等代码块
    for m in re.finditer(r"```[^\n]*\s*(.*?)```", text, re.S):
        candidates.append(m.group(1).strip())

    # 2. 扫描裸 JSON：从 { 或 [ 开始，按平衡括号截取
    for open_ch, close_ch in (("{", "}"), ("[", "]")):
        i = text.find(open_ch)
        while i != -1:
            depth = 0
            in_str = False
            esc = False
            for j in range(i, len(text)):
                c = text[j]
                if in_str:
                    if esc:
                        esc = False
                    elif c == "\\":
                        esc = True
                    elif c == '"':
                        in_str = False
                else:
                    if c == '"':
                        in_str = True
                    elif c == open_ch:
                        depth += 1
                    elif c == close_ch:
                        depth -= 1
                        if depth == 0:
                            candidates.append(text[i:j + 1])
                            break
            i = text.find(open_ch, i + 1)
    return candidates
~~~

第二步是清洗与低风险修复。去 BOM、去尾逗号、替换 `True/False/None` 为 JSON 的 `true/false/null`，必要时用 `json5` 处理注释和单引号。尽量不要做“自动补括号”这种高风险修复，因为补错位置会掩盖真实问题。

第三步是 schema 筛选。多个候选块时，不要盲目取第一个。比如需要 `action` 字段，就优先返回包含该字段且能通过 `pydantic` 或 `jsonschema` 校验的候选。示例块通常很短，真实指令往往字段更完整、嵌套更深，必要时按长度或字段完整度打分。

第四步是失败回传。解析失败后，把截断后的原始输出和错误信息回传给模型，让它重新生成。重试最多 1～2 次，同时记录完整 raw 输出用于追踪。

## 踩坑点

1. **字符串里的括号会干扰平衡扫描**。扫描裸 JSON 时必须维护 `in_str` 和 `escape` 状态，否则字符串里的 `}` 会导致截断。
2. **多个 JSON 块不一定是“第一个才对”**。模型经常先给一个示例，再给真实结果。只取第一块会让后续动作跑错。
3. **自动补括号很危险**。缺一个 `}` 时补全会变成合法 JSON，但语义可能已经变了。低风险修复只做尾逗号、BOM、注释、布尔值替换；高风险修复必须告警并保留原始输出。
4. **代码块不一定是 JSON**。可能是 JavaScript object 或 Python dict，需要先经过清洗；如果直接 `json.loads`，`{'x': 10}` 会失败。
5. **大输出性能问题**。对超长日志反复扫描可能拖慢流程。可以限制单次扫描长度、候选数量，或只扫描前 N 个优先级区域。

## 可复用建议

- 在 OpenClaw 的统一出口处挂解析器：工具调度前、MCP 参数解码、插件返回值处理都走 `parse_llm_payload`，避免每个插件自己写一套。
- 保留 raw 输出日志，至少保留失败样本。混合格式不是模型错误，而是环境噪声，你需要看到原始噪声才能优化。
- 代码库中维护一个混合输出样本集：前导文字、尾随文字、代码块、多 JSON 块、尾逗号、Python dict 风格、注释、`null/None` 混用等。用这些样本做回归测试。
- 区分线上和调试模式：线上默认开启低风险修复，调试时开启严格模式，直接暴露原始输出，方便定位模型行为。
- 失败重试时不要把完整长日志塞回模型，只给截断片段和结构化错误信息，节省 token。

## 总结

防御性解析不是在让模型迁就你，而是承认：只要 Agent 链路里存在不可控的外部输出，解析层就必须把它当成不可信输入。JSON 标签格式混合处理，本质是候选提取 + 低风险修复 + schema 筛选 + 失败回传。这套思路在 OpenClaw、MCP、插件自动化和 Agent 工具调用里都能复用。解析器越稳，整条自动化链路越不容易被一次脏输出打挂。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/f85c9f8e2eb99891.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/52c1b32f80f29e9f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/c93b5146e2031096.png)

