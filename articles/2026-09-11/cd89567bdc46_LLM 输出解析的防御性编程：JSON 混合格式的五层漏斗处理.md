---
title: LLM 输出解析的防御性编程：JSON 混合格式的五层漏斗处理
feedId: 37009
source: 综合讨论
publishedAt: 2026-09-11
---

## 背景

做 Agent / 插件 / 自动化流水线时，我们经常要求模型「只输出 JSON」。提示词写得再严，线上跑几天就会发现：同一个 prompt，模型有时给裸 JSON，有时包一层 ```json 围栏，有时套上 `<result>...</result>` 这类自造标签，前后还可能带一句「以下是解析结果」。只要下游写的是 `json.loads(resp)`，这些变体都会变成异常或静默失败。

核心观点一句话：**LLM 输出应视为不可信输入，解析层必须像处理用户输入一样做防御。**

## 问题拆解

实际遇到的脏输出大致四类：

1. **围栏包裹**：```json ... ```，偶尔还嵌在普通代码块里；
2. **自定义标签**：`<json>`、`<output>` 等模型自创的「结构化」；
3. **混入自然语言**：前导说明、后置道歉（「抱歉，正确格式如下」）；
4. **语法瑕疵**：尾逗号、单引号、全角引号/逗号、BOM、`//` 注释。

## 做法：五层漏斗式解析

```python
import json, re

def parse_llm_json(text: str):
    s = text.strip().lstrip('\ufeff')

    # 1. 直接解析
    try:
        return json.loads(s)
    except json.JSONDecodeError:
        pass

    # 2. 剥围栏 / 自定义标签
    m = re.search(r'```(?:json)?\s*(.*?)```', s, re.S) \
        or re.search(r'<(?:json|result|output)>(.*?)</\1?>', s, re.S)
    if m:
        try:
            return json.loads(m.group(1).strip())
        except json.JSONDecodeError:
            s = m.group(1)

    # 3. 括号配对截取（栈匹配 {} / []，不是贪婪正则）
    s = extract_balanced(s)

    # 4. 宽松修复：全角转半角、去尾逗号，兜底交给 json_repair
    s = pre_clean(s)
    try:
        return json.loads(s)
    except Exception:
        log.warning("repair path hit")   # 修复路径必须留痕
        return json_repair.loads(s)
```

三个要点：

- **括号配对要用栈**：`{.*}` 这类正则会把前后无关文本一起吞进来，字符串里含 `{` 时必挂；
- **解析成功 ≠ 数据正确**：拿到 dict 后再过一层 pydantic / JSON Schema 校验字段和类型；
- **修复路径必须留痕**：打了哪个补丁、原始输出存档，方便离线统计修复率、迭代 prompt。

## 踩坑点

1. **别迷信 temperature=0**。低温只是降低漂移概率，不提供格式保证。结构化输出 / tool call 参数能用就用，但兜底解析照样要写。
2. **json_repair 不是免费的**：它可能把 `"1,234"` 修成 `1234`、给缺失字段补默认值，语义悄悄变了。修复成功也要标记 review。
3. **重试要有预算**：解析失败后带上「上次原样输出 + 错误信息」回炉，最多 1–2 次，否则坏循环会持续烧 token。
4. **全角字符**是中文场景高发坑：`，`、`：`、`“”` 混进 JSON 里 `json.loads` 直接炸，pre_clean 里统一替换。
5. **流式场景**别拿半截 JSON 去 loads，要么攒齐再解析，要么换增量解析器。

## 可复用建议

- 把漏斗封装成统一的 `parse_llm_json()` 入口，团队别各写一版正则；
- 建「脏输出语料库」：每次线上解析失败，匿名化后入库当单元测试用例，回归测试比新技巧更有价值；
- 埋两个指标分开看：**一次解析成功率**反映 prompt 质量，**修复后成功率**反映解析层兜底能力——才知道该优化哪一头。

## 总结

Prompt 约束和解析兜底是两道独立的保险，缺一不可：前者提高「输出是干净 JSON」的概率，后者保证「即使不干净，系统也不炸」。把 LLM 输出当不可信输入来写代码，是 Agent 工程里最便宜也最划算的一笔投资。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/5d62f36e18e82c68.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/75282f04c16bed0b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/cf6358604635094b.png)

