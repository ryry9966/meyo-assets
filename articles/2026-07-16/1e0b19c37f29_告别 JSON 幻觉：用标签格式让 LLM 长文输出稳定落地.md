---
title: 告别 JSON 幻觉：用标签格式让 LLM 长文输出稳定落地
feedId: 29238
source: 综合讨论
publishedAt: 2026-07-16
---

## 问题背景

在 OpenClaw 这类 Agent 编排框架中，我们经常让 LLM 生成结构化的长文本——比如自动化报告、多段落摘要、或者插件调用返回的复杂数据。常规做法是要求模型输出 JSON，然后在管道中解析。但当输出超过几百 token，JSON 就越来越容易“坏掉”：

- 对象嵌套过深，花括号匹配错误
- 正文中换行、引号未正确转义
- 模型在流式输出中途改变了 JSON 结构
- 数组缺失逗号，或者多了一个尾随逗号，JSON.parse 直接报错

最让人头疼的是，这些错误常常发生在管道末端——解析即将完成时崩溃，前面的输出全部作废。对于自动化流水线来说，一座稳健的桥梁远比一座偶尔断裂的玻璃桥有价值。

## 核心思路：用显式标签替代隐式语法

一种在工程实践中逐渐沉淀下来的思路是：放弃要求模型严守 JSON 语法，转而使用 **自定义的文本标签格式**。例如，对于一篇包含标题、摘要、章节和尾注的报告，可以要求模型输出：

```
<TITLE>系统运维周报</TITLE>
<SUMMARY>本周完成灰度发布3次，故障处理2起...</SUMMARY>
<SECTION name="发布详情">
周一发布订单服务 v2.1.0，灰度比例 10%
</SECTION>
<SECTION name="故障记录">
故障 ID：INC-2024-087，根因：连接池耗尽...
</SECTION>
<FOOTNOTES>
- 下周一全量发布
- 计划升级 Redis 集群
</FOOTNOTES>
```

这本质上是一种“文档标记语言”，利用 LLM 本身对 XML/HTML 标签格式的良好跟随能力，把结构化约束降级为简单的文本边界识别。

## 具体做法

### 1. 设计标签协议

标签设计需遵循几个原则：

- **扁平化**：尽量避免嵌套。比如用多个 `<SECTION>` 标签并列，而不是一个大数组嵌套对象。
- **白名单闭合**：明确规定每个标签是自闭合、还是需要结束标签。复杂内容用包裹标签，简单字段可用自闭合占位（如 `<ID/>` 但模型可能忽略斜杠，需谨慎）。
- **保留字约定**：标签名使用全大写英文，减少与正文冲突。例如 `<SUMMARY>` 而不是 `<summary>`。

### 2. 修改 System Prompt 与 Few-shot

在提示词中，不要只写“请用 JSON 返回”，而是给出清晰的标签模板。提供 1~2 个完整示例，展示开始和结束标签，以及标签内部的自由文本格式。同时明确告诉模型：

- 不需要转义内部引号和换行
- 标签必须成对出现
- 不要在标签之外输出解释性文字，除非用我们指定的注释标签（如 `<NOTE>`）

举例 System Prompt 片段：

> 你输出的内容将直接进入自动化解析流水线。请严格使用以下标签格式，标签名必须完全一致：  
> `<REPORT>` 包裹整个输出  
> `<TITLE>` 标题  
> `<SUMMARY>` 摘要  
> `<SECTION name="...">` 每个章节  
> `<FOOTNOTES>` 尾注（可选）

### 3. 解析器实现

解析侧放弃通用 JSON 解析，改为轻量级文本提取器。以 Python 为例：

```python
import re

def parse_report(text: str):
    # 提取 REPORT 包裹的内容
    report_match = re.search(r'<REPORT>(.*?)</REPORT>', text, re.DOTALL)
    if not report_match:
        raise ValueError("Missing <REPORT> block")
    content = report_match.group(1)

    # 提取各个标签
    title = re.search(r'<TITLE>(.*?)</TITLE>', content, re.DOTALL)
    summary = re.search(r'<SUMMARY>(.*?)</SUMMARY>', content, re.DOTALL)
    sections = re.findall(r'<SECTION name="(.*?)">(.*?)</SECTION>', content, re.DOTALL)
    footnotes = re.search(r'<FOOTNOTES>(.*?)</FOOTNOTES>', content, re.DOTALL)

    return {
        "title": title.group(1).strip() if title else "",
        "summary": summary.group(1).strip() if summary else "",
        "sections": [{"name": s[0], "content": s[1].strip()} for s in sections],
        "footnotes": footnotes.group(1).strip() if footnotes else ""
    }
```

比起 JSON 解析，正则匹配对不完美格式的容忍度极高——即使模型中缺失了某个闭合标签，我们也能对已提取的部分进行缺省处理，而不会直接整个输出报废。

### 4. 错误恢复与流式适配

在 OpenClaw 的流式回调中，可以实时累积文本，每隔 N 个 token 尝试检测标签闭合。一旦发现某个关键标签闭合，就可以安全地消费该部分内容，而无需等待整个输出结束。这大大提升了 Agent 管道的时效性。

对于未闭合标签，可增加一个“温柔修复”步骤：在文本末尾追加缺失的结束标签，再重新解析。简单的启发式——统计每个标签的开始和结束次数，自动补齐——就能救回约 80% 的不完整输出。

## 踩坑点

1. **标签名冲突**：如果正文中会出现类似 `<TITLE>` 的文本，需要约定转义规则（如模型必须输出 `&lt;TITLE&gt;`）或使用生僻标签名（如 `<RX_TITLE>`）。实践中更省事的办法是要求模型在普通文本中不要使用尖括号，虽然不够严谨，但多数场景够用。

2. **嵌套关系丢失**：标签格式天然不擅长表现深层嵌套。如果需要展示复杂层级，可以考虑用属性标识路径，如 `<SECTION level="1" path="/group/subgroup">`，但此时已经偏离初衷，变得接近重新发明 JSON。所以只推荐用于浅层结构化输出。

3. **模型“创意”溢出**：有些模型会在标签外添加礼貌性结语（“以上是报告内容，希望对您有帮助”）。需要在提示词中再三强调，并在解析时明确裁剪掉 `<REPORT>` 之外的内容。

## 可复用建议

- **先评估，后迁移**：如果当前 JSON 输出成功率高于 95%，无需切换。但一旦进入长文本（>2000 token）或自由文本较多的场景，标签方案值得立刻引入。
- **双通道输出**：在过渡期，可以让模型同时输出 JSON 和标签格式，用一个小解析器探测哪种成功了就用哪种，逐步建立信心。
- **把解析逻辑下沉为 OpenClaw 插件**：将标签提取器封装为 Agent 可调用工具，让后续节点直接消费结构化 dict，保持编排简洁。
- **监控标签闭合率**：在日志中记录每次输出的标签闭合情况，逐步优化提示词和 few-shot，将其作为模型选择的评估指标之一。

## 总结

让 LLM 输出精确 JSON 是一件需要运气的事，但当我们将重心从“教模型写正确的 JSON”转移到“给模型一种更鲁棒的约束方式”，整个自动化管道的稳定性会有肉眼可见的提升。标签格式不是银弹，但在长文、多段落、自由文本与结构化混合的场景下，它用简单的工程约束换来大幅降低的解析失败率。对于注重稳定输出的 OpenClaw 实践者而言，这是一次非常务实的设计转向。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/8f799f7e65140ee4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/73409570b2b57a41.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/cc641f8a7b028163.png)

