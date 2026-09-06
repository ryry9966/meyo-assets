---
title: 让主 Agent 学会派活：subagent 并行编排的实践与踩坑
feedId: 36386
source: 综合讨论
publishedAt: 2026-09-07
---

## 背景

单 Agent 循环适合线性任务，但一旦遇到“对 20 个模块各跑一遍检查”这类可拆分的批量工作，串行执行的问题就暴露出来：耗时长、中间输出把主上下文撑爆、用户只能干等。subagent 编排的思路很朴素：主 Agent 只做拆解和汇总，把相互独立的子任务派给并行的子 Agent，各自在隔离上下文里跑完，只把结论带回来。

## 问题

我们在自动化项目里遇到三个典型症状：

1. 主上下文被工具原始输出污染，几轮之后模型开始“忘事”；
2. 批量任务串行跑，一次 10 分钟的扫描占满整个会话；
3. 主 Agent 一边干活一边记状态，提示词越写越乱。

## 做法

我们的编排结构分四步：

1. **定义子任务接口**。每个 subagent 固定三要素：任务描述、允许的工具白名单、输出格式（约定 JSON：status / findings / artifact 路径）。
2. **主 Agent 拆解派活**。通过一个 task 工具（或 MCP 暴露的 spawn 接口）发起 N 个子任务，每个子 Agent 拥有独立上下文，互不可见。
3. **约束产出方式**。子 Agent 把大产物（报告、diff、抓取结果）写到磁盘或对象存储，返回值只带路径和摘要，不贴全文。
4. **主 Agent 汇总**。收齐结果后统一判断，失败的子任务记录原因，不中断整批。

伪代码大致是：

```python
results = pool.map(run_subagent, subtasks, max_workers=4)
for r in results:
    if r.status != "ok":
        log(r.error)
summary = main_agent.aggregate([r.digest for r in results])
```

## 踩坑点

- **返回值太肥**。最初让子 Agent 直接返回完整分析，四个并行就把主上下文打满。改成“摘要 + 文件路径”后立刻可控。
- **并行写冲突**。两个子 Agent 同时改同一批文件，后写的覆盖先写的。要么按文件分区，要么子 Agent 只读、由主 Agent 统一落盘。
- **成本翻倍**。并行 5 个子 Agent 意味着 token 消耗接近 5 倍，先在小批量上估算再放开。
- **错误被吞**。某个子 Agent 静默失败，主 Agent 拿到空结果当成“无问题”。必须强制 status 字段，聚合时显式检查。
- **过度拆分**。有依赖关系的任务硬拆并行，只会带来返工。拆之前先问一句：这些子任务真的互不依赖吗？

## 可复用建议

- 子 Agent 保持无状态、单一职责，一个子任务只干一件事；
- 并发上限从 3–5 起步，压测后再调；
- 批量、可枚举的任务用脚本做确定性 fan-out，比让模型自己决定拆几个更稳；
- 所有产物落盘，上下文里只留路径和一句话摘要；
- 每个子任务设超时和一次重试，避免僵尸任务挂满队列。

## 总结

subagent 编排本身不难，难的是约束：约束返回值大小、约束写权限、约束并发数。把这三条纪律立住，主 Agent 的上下文干净了，批量任务的墙钟时间也能从串行的 N 倍压到接近 1 倍。建议从“批量只读分析”这类最安全的场景开始试，跑顺了再放开写操作。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/c21440ce57f20415.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/fc11785f58fdf36f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/7527ccd80e37d02e.png)

