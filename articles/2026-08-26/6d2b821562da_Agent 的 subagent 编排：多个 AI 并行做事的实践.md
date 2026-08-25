---
title: Agent 的 subagent 编排：多个 AI 并行做事的实践
feedId: 34786
source: 综合讨论
publishedAt: 2026-08-26
---

## 背景

单个 Agent 处理复杂任务时，常见问题是：上下文越来越长、工具调用互相干扰、串行执行太慢。比如一个任务要同时查资料、读代码、做数据提取，如果都塞进一个会话里，模型容易在中间步骤“迷路”，token 消耗也很快。

更工程化的做法是：把任务拆成多个 subagent，每个 subagent 只负责一个窄任务，并行执行，最后回收结果。这样上下文隔离、工具权限隔离，主 Agent 只需要做三件事：拆解、派发、汇总。

在 OpenClaw 这类环境里，subagent 通常可以通过子会话、内置 agent 工具或 MCP worker 来实现。下面分享一套实践方式，不绑定具体框架，但可以在 OpenClaw/Agent/MCP 场景下直接套用。

## 问题

不是所有任务都适合用 subagent。适合的场景有几类：

1. **多源并行采集**：同时查多个数据源、多份文档、多个 API。
2. **多文件/多模块分析**：一个仓库里不同目录可以独立检查。
3. **权限隔离**：某个 subagent 只给只读工具，另一个才给写工具。
4. **上下文隔离**：避免长文本互相污染。

真正难的不是“能不能并行”，而是：调度开销、结果合并、失败重试、成本控制。并行数量一多，主 Agent 的汇总压力会陡增，甚至产生冲突或幻觉。

## 做法/步骤

### 1. 先定义主 Agent 与 subagent 的边界

主 Agent 不要做具体执行，只做：

- 拆解任务
- 选择 subagent
- 合并结果
- 仲裁冲突

subagent 必须收窄目标。每个 subagent 最好能用一个短句描述清楚，例如：

```text
doc-research：根据给定主题，从指定数据源收集 3~5 条相关材料，输出 JSON。
```

如果某个 subagent 的任务写出来还是“分析项目并给出建议”，说明拆得不够细。

### 2. 给 subagent 一份“工作说明书”

每个 subagent 需要明确：

- 目标
- 输入格式
- 可用工具
- 输出 schema
- 失败标准
- 最大 token 或时间限制

一个示意配置可以这样写：

```yaml
name: doc-research
tools: [mcp.search, mcp.fetch]
input:
  topic: string
  max_sources: int
output:
  summary: string
  items:
    - title: string
      url: string
      snippet: string
limits:
  max_tokens: 3000
  timeout_sec: 90
```

字段名不必照搬，关键是让 subagent 知道自己必须“返回结构化结果”，而不是自由发挥。

### 3. 并行控制

并行数不是越高越好。外部 API 限流、模型上下文切换、主 Agent 汇总压力都会限制实际收益。一般来说，先跑 2~3 个并发，稳定后再调到 4~5 个。

可以用一个简单的调度逻辑：

```text
tasks = [task_a, task_b, task_c, task_d]
results = parallel_map(
    tasks,
    max_workers=3,
    timeout=120,
    retry=1,
)
```

失败的 subagent 不要无脑重试。先看失败原因：超时可以重试一次；工具权限不足就直接终止，避免重复烧 token。

### 4. 结果回收与合并

subagent 的输出尽量要求 JSON 或带结构的 Markdown。主 Agent 合并时，重点做三件事：

- 去重
- 冲突检测
- 证据保留

如果两个 subagent 给出相反结论，不要让主 Agent 自己“感觉”哪个对。可以让它退回给其中一个 subagent 补充证据，或者标记为“未解决”。

### 5. 在 OpenClaw 环境里落地

如果你的环境支持 MCP，可以把 subagent 封装成 MCP tool，主 Agent 通过工具调用。比如：

```text
tools:
  - run_research_subagent
  - run_code_review_subagent
  - run_data_extract_subagent
```

每个 subagent 内部有独立 system prompt、独立上下文窗口、独立工具集。主 Agent 只拿到最终返回值，不接触中间过程。

## 踩坑点

### 1. 上下文隔离不等于约束隔离

subagent 很容易“跑偏”，因为主 Agent 的全局约束没有传给它们。共享约束比如“不要修改生产数据”“不要输出猜测性结论”，必须写进每个 subagent 的 prompt 里。

### 2. 并行调用同一 API 触发限流

多个 subagent 同时请求同一个搜索接口或同一个数据库，可能触发限流。给 subagent 配置不同数据源，或者给并发加间隔。

### 3. 结果合并时主 Agent 和稀泥

一旦多个 subagent 意见不一致，主 Agent 容易输出“可能 A 对，也可能 B 对”。更好的做法是：要求 subagent 返回 source、confidence、evidence，合并时只保留有证据支撑的结论。

### 4. 工具权限过大

subagent 拿到写权限后，可能做多余动作。比如一个只负责读文档的 subagent，不应该有 `write_file` 工具。最小权限原则在 subagent 里更重要。

### 5. 递归 subagent 导致爆炸

如果 subagent 还能再创建 subagent，层级会失控。限制深度最多 1~2 层，主 Agent 下面一层 subagent 就好。

## 可复用建议

- **从 2~3 个 subagent 开始**，先跑通一条小链路。
- **把 subagent 当纯函数用**：同一输入尽量得到相似输出，少依赖全局状态。
- **给每个 subagent 稳定 ID 和日志**，方便回溯哪一步出错。
- **用 max_tokens 和 timeout 兜底**，避免单个 subagent 拖垮整条流程。
- **保留主 Agent 的最终决策权**，subagent 提供候选方案，主 Agent 做 trade-off。

## 总结

subagent 编排解决的是“复杂任务怎么拆分”和“多个执行体怎么协作”的问题。它不是银弹，适合可分治、边界清晰、结果可结构化合并的任务。

真正带来收益的不是“并行数量”，而是：上下文隔离、结构化回收、并发控制、冲突仲裁、可观测性。把这五件事做好，多个 AI 并行做事才会从“看起来很酷”变成“确实可用”。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/14b6212cae794ba2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/2be7d08881f9b4be.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/1f7715742c194b3a.png)

