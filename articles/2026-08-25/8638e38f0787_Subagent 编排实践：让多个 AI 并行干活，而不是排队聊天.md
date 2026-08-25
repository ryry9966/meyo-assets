---
title: Subagent 编排实践：让多个 AI 并行干活，而不是排队聊天
feedId: 34693
source: 综合讨论
publishedAt: 2026-08-25
---

## 背景

单个 agent 处理复杂自动化任务时，很容易出现三种问题：上下文越滚越长、多步骤串行耗时、不同工具之间互相干扰。比如一个 agent 既要从 MCP 数据源采集信息，又要做代码检查，还要跑测试；它会在不同任务之间反复切换，任何一个步骤异常都会拖垮整条链路。

subagent 编排的思路是把主 agent 从“执行者”变成“调度者”：主 agent 负责任务拆分、结果汇总和异常处理，具体工作拆给多个 subagent 并行执行。OpenClaw 里的 subagent 机制适合做这类隔离执行，尤其适合 MCP 工具和插件已经比较多的场景。

## 问题

不是所有任务都适合并行。真正值得用 subagent 的，通常满足三个条件：

1. 任务边界清晰，子任务之间没有强依赖；
2. 每个子任务只读共享资源，写操作收敛到主 agent；
3. 子任务输出可以被压缩成结构化结果，而不是大段原文。

如果任务之间相互依赖、需要实时共享中间状态，subagent 并行反而会增加协调成本。

## 做法

### 1. 拆分任务，定义合同

主 agent 只做三件事：拆分、分发、汇总。每个 subagent 必须有明确输入、允许使用的工具、输出 schema 和停止条件。

例如一个“仓库健康检查”自动化，我会拆成三个 subagent：

- `diff-review`：读取 PR diff，返回风险列表；
- `test-runner`：运行测试，返回失败用例摘要；
- `doc-check`：检查文档和变更是否同步，返回缺失项。

每个 subagent 都要求输出 JSON，字段固定，且限制长度。例如：

```json
{
  "status": "ok",
  "summary": "3 个风险点",
  "items": ["...", "..."]
}
```

不要直接返回几百行日志或完整 diff。

### 2. 配置子任务

不同版本/运行时的字段可能有差异，这里只描述通用配置思路。关键不是 prompt 写多长，而是给足约束：

- 角色：一句话说明自己只做什么；
- 工具白名单：只给必要工具，避免误调用写操作；
- 输出 schema：给出 JSON 字段说明；
- 停止条件：找到结论就返回，不要继续探索；
- 超时与 token 上限：每个 subagent 强制限制。

伪代码示意：

```python
tasks = [
    spawn("diff-review", prompt_task, tools=["read-diff"], timeout=120),
    spawn("test-runner", prompt_task, tools=["run-tests"], timeout=300),
    spawn("doc-check", prompt_task, tools=["read-docs"], timeout=120),
]
results = await gather(tasks)
summary = main_agent.aggregate(results)
```

主 agent 不应该把所有 subagent 原始输出再读一遍。我会让每个 subagent 在返回前做一次“压缩”，只带回摘要和结构化数据。主 agent 拿到的是精简结果，而不是完整执行轨迹。

### 3. 合并结果与异常处理

主 agent 汇总时不需要重新推理，只需要做三件事：

- 检查状态：哪些成功、哪些失败、哪些超时；
- 冲突标记：如果两个 subagent 对同一事实有矛盾结论，标记出来，不强行融合；
- 生成最终报告：面向用户的摘要，而不是内部日志。

对于一个失败或超时的 subagent，可以设置重试一次。如果第二次仍失败，就返回固定错误对象，让主 agent 决定是否降级处理。

## 踩坑点

### 1. 上下文复制成本

subagent 通常会继承一部分主 agent 上下文。如果主 agent 携带了大段历史消息，并行多个 subagent 会导致上下文成倍膨胀。解决方法是给 subagent 传最小输入：只要任务目标、必要文件路径和 schema，不要传完整聊天记录。

### 2. 工具权限过大

并行 worker 越多，越容易误调用危险工具，尤其是写文件、发消息、执行 shell 的子任务。建议 subagent 默认只读，写操作由主 agent 统一执行。如果必须写，使用独立目录或命名空间，避免互相覆盖。

### 3. 结果格式漂移

模型输出 JSON 不稳定是常见问题。不要假设一次就能得到完美 schema。可以在 subagent 返回后做校验，格式错误就自动重试一次，并附带错误信息。两次都失败则返回 `{"status": "error"}`，不要无限重试。

### 4. 并发不是越多越好

底层模型有速率限制，MCP server 也可能有连接限制。并行 2-4 个 subagent 通常是最稳的区间。超过 6 个，收益下降且限流概率上升。可以先从 2 个并行开始，稳定后再增加。

### 5. MCP 会话冲突

如果多个 subagent 同时访问同一个 MCP server，注意会话和连接管理。有的 MCP server 不支持并发会话，或共享同一连接会导致状态串扰。若遇到诡异结果，可以给每个 subagent 配置独立连接，或把该任务改成串行。

## 可复用建议

- 先串行后并行：新任务先让单个 subagent 跑通，确认输出稳定后再并行。
- 固定模板：把 subagent 定义成可复用任务文件，包括角色、工具白名单、schema、超时、重试次数。例如 `repo-health-diff-review.task.json`。
- 控制并行度：默认 3 个并发，保留一个余量给主 agent 处理异常。
- 记录 token 和耗时：每个 subagent 返回时带上 `elapsed_ms` 和 `tokens_used`，方便定位是哪个步骤又慢又贵。
- 主 agent 保持轻量：主 agent 上下文里不要放执行细节，只放任务定义和汇总结果。执行细节放在 subagent 内部。
- 为结果设置硬上限：例如每个 subagent 输出不超过 500 字或 20 行，否则强制截断或压缩。

## 总结

subagent 编排的核心不是“让更多 AI 一起聊天”，而是用短上下文 worker 做隔离执行，用轻量主控做调度和汇总。它能降低单 agent 的上下文压力、提升多任务吞吐，但也带来权限、格式、限流和合并成本。

可落地的路径是：先识别 1-2 个可并行的旧任务，定义好输出 schema，用最低权限的工具白名单跑通串行版本，再改成 2-3 个并行。跑稳之后，再考虑把更多任务拆出去。这样得到的不是演示效果，而是一个可以重复运行、能排查、能限流的生产级编排。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/68e917b9fe997a87.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/428408bcddf2bc2a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/1236821f5be0abc0.png)

