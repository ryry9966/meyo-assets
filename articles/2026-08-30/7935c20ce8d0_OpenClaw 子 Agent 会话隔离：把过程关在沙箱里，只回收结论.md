---
title: OpenClaw 子 Agent 会话隔离：把过程关在沙箱里，只回收结论
feedId: 35422
source: 综合讨论
publishedAt: 2026-08-30
---

## 背景

OpenClaw 主会话承担计划、工具选择、上下文维护等关键职责。引入子 Agent 后，主会话很容易变成“噪声接收器”：子 Agent 的长推理、错误重试、工具日志、中间 JSON、异常栈一股脑回流。结果通常是主 Agent 后续调用开始复述无关内容、丢失原始目标、token 成本暴涨。

session 隔离要解决的不是“不让子 Agent 访问上下文”，而是控制：**什么信息以什么形状回到主会话**。

## 问题

最常见的污染有两类。

1. **上下文污染**：子 Agent 返回完整 transcript 或工具输出，主会话上下文被大量中间态占据。  
2. **状态污染**：子 Agent 直接写文件、改环境变量、写全局 memory，主会话后续步骤基于被修改的状态继续运行。

表现通常是：主会话越跑越偏；重试时无法回到调用前状态；多子任务并发时相互覆盖。

## 做法 / 步骤

### 1. 子 Agent 只接收最小输入

不要给完整历史。只给任务目标、必要输入、约束和输出 schema。主会话的上下文不是子 Agent 的工作台。

### 2. 使用独立 session 或 fork

主会话只保留一条 delegation 摘要。子 Agent 使用独立 session_id，配置 `max_turns` 和 token budget，避免无限扩展。

```yaml
subagent:
  session_policy: fork
  context_mode: minimal
  max_turns: 12
  max_return_tokens: 2000
```

### 3. 回传只收 structured envelope

在 OpenClaw 编排层约定子 Agent 最终返回结构化结果，而不是自由文本：

```yaml
return:
  mode: structured_summary
  schema:
    status: string
    conclusion: string
    artifacts: list
    blockers: list
    evidence: list
```

中间过程不回传。工具输出只保留关键证据链接或文件路径。

### 4. 工具副作用隔离

写操作默认走 patch/diff，主会话审核后再 apply。环境变量和全局 memory 设为只读。子 Agent 只能返回“建议怎么改”，不能直接修改主会话状态。

### 5. 回传后做校验与压缩

主会话侧检查必填字段、长度限制、artifact 路径是否存在。校验失败就丢弃详细回传，只记录失败原因，不把原始输出拼进上下文。

### 6. 保留回滚点

每次子 Agent 调用前记录主 session checkpoint。出现污染时可以直接回滚到调用前，而不是继续带病运行。

## 踩坑点

- **把“总结”当成事实**：总结可能丢失关键条件，主 Agent 据此做出错误决策。需要 `evidence` 字段支撑结论。
- **只做上下文隔离，忽略副作用**：子 Agent 写入文件或修改环境变量后，即使不返回内容，主会话也已经被污染。
- **给子 Agent 过多上下文**：子 Agent 上下文越长越容易跑偏，回传摘要也越可能带回噪声。
- **JSON 返回未截断或转义**：长 JSON 直接插入 prompt，可能撑爆上下文或破坏格式。
- **子 Agent 会话未回收**：旧 session 留在 memory/向量库里，后续检索可能召回过期信息。

## 可复用建议

- 默认最小上下文：目标 + 输入 + 约束 + 输出 schema。
- 统一 envelope：所有子 Agent 返回 `status / conclusion / artifacts / blockers / evidence`，不加自由文本。
- 主会话维护不可变 delegation log：任务描述、时间、返回摘要、hash，不保留原始中间输出。
- 工具层做只读/写审核分级，非只读操作必须走 patch 或 dry-run。
- 增加污染检查：子任务后让主会话回答“你刚才是否修改了 X？”这类验证，判断状态是否越界。
- 设置回传长度上限和字段白名单，超限截断或直接丢弃。

## 总结

OpenClaw 的 session 隔离不是不让子 Agent 思考，而是让主会话只接收可审计、可压缩、可回滚的结果。子 Agent 可以跑得很野，但返回主会话的边界必须窄。这样多个子 Agent 才能安全并发，主会话也不会因为一次长任务而报废。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/dfec9be7d5cfc174.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/4349885c1dc6e7fa.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/8f17c043c37385cb.png)

