---
title: LLM 输出解析的防御性编程：一套处理 JSON 混合包装的分层管线
feedId: 35932
source: 综合讨论
publishedAt: 2026-09-03
---

## 背景

OpenClaw 的插件和 MCP 工具链里，很多环节依赖 LLM 返回结构化数据：意图路由、参数抽取、工具结果汇总。我们习惯在 prompt 里写一句"只输出 JSON"，然后就当它一定会守约。但只要接的不是单一模型——尤其是通过 MCP 挂了多家第三方 provider 之后——同一个 prompt 拿回来的包装方式差异会大得超出预期。解析器只覆盖一种格式，上线后迟早会在某个模型、某次长输出上翻车。

## 问题：同一个字段，五种壳

生产日志里实际见过的变体：

1. 裸 JSON，直接可解析；
2. ` ```json ` 围栏包裹；
3. `<json>...</json>`或模型自创的标签包裹；
4. 前后各一段解释文字，中间夹一段 JSON；
5. 输出被 `max_tokens` 截断，JSON 根本不完整。

## 做法：定位靠正则，解析靠状态机，验收靠 schema

核心思路是把解析做成一条有序的策略链，任何一环命中就交给 schema 校验：

```python
def extract_json(text: str):
    t = preprocess(text)              # 去 BOM/零宽字符，不动引号
    for fn in (try_direct,            # 1) 直接 json.loads
               try_fenced,            # 2) ```json 围栏，从后往前找
               try_tags,              # 3) <json> 类标签，取最后一个匹配
               try_braces,            # 4) 括号平衡扫描（感知字符串/转义）
               try_repaired):         # 5) 去尾逗号和注释后重试
        result, strategy = fn(t)
        if result is not None:
            return validate(result), strategy   # pydantic 校验 + 信封解包
    raise ParseError(raw=text)
```

几个关键点：

- **括号平衡扫描必须带字符串状态**（`in_string` / `escape`），否则字段值里出现 `"{name}"` 这类模板串就会切错位置。
- **标签匹配用非贪婪并取最后一个匹配**——模型经常把 prompt 里的示例原样复读一遍，第一个匹配往往是你的指令示例。
- **解析成功不等于数据可用**：必须过 schema 校验，并处理信封结构，比如模型好心包了一层 `{"result": {...}}`。
- **记录 `(value, strategy, raw)`**：线上观察各策略命中率，格式漂移会先体现在命中率变化上，而不是先变成故障。

## 踩坑点

1. **不要全局替换全角引号/智能引号**。中文正文里全角引号是合法内容，全局替换会直接改坏数据。只在结构解析失败且上下文模式明确时定向处理。
2. **去尾逗号、去注释同样要带字符串状态**，否则会改写字段值里的合法逗号和斜杠。
3. **截断的输出别试图"补括号"救回来**。先看 `finish_reason`，截断就走缩短 schema、分步输出这条路；硬补大概率得到语法合法但语义错误的 payload，比解析失败更危险。
4. **正则只负责定位候选区间**，永远不要用正则做完整 JSON 的结构切分。
5. 修复型重试（把报错和原文丢回给模型改）有效但要设预算：只重试一次，失败就抛结构化错误，别返回空字典静默过关。

## 可复用建议

- 解析逻辑收敛到一个共享模块，各插件不要各自手写 `find("{")`；
- 策略链做成有序配置，方便按 provider 微调顺序；
- 把线上真实的"坏输出"脱敏后沉淀成 fixture 库，在 CI 里回放，比单元测试里拍脑袋构造的边界 case 有用得多；
- 运行时支持 JSON Schema 约束或 function call 时优先用原生通道，这套解析器定位是第三方模型的兜底，而不是首选。

## 总结

LLM 输出解析没有银弹，但"预处理 → 多策略提取 → schema 验收 → 观测回灌"这条分层管线，能把格式漂移从线上事故降级成一个可度量的指标。原则只有一条：别相信"只输出 JSON"，设计时假设它一定会破约，然后让破约变得无害。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/1d8952c9b26db6d2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/5fbfb627fd7b22b5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/698ce95d15f5d060.png)

