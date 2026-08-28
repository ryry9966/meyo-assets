---
title: Agent 的 subagent 编排：多个 AI 并行做事的实践
feedId: 35025
source: 综合讨论
publishedAt: 2026-08-28
---

# Agent 的 subagent 编排：多个 AI 并行做事的实践

## 背景

单个 agent 在处理“调研 10 个数据源”“批量分析 20 个仓库”这类任务时，串行执行通常有两个瓶颈：一是总耗时会线性叠加，二是前面任务产生的中间文本会不断进入上下文，越往后噪声越多。OpenClaw 的 subagent 能力允许主 agent 把独立子任务派发给多个子 agent 并行处理，再回收结构化结果。这个模式看起来简单，实际落地时问题往往不在“能不能并行”，而在“结果能不能可靠地收回来”。

## 问题

并行编排主要面对三类问题：

1. 上下文隔离：主 agent 如果把自己上下文里的所有内容都转给 subagent，子任务会被无关信息干扰。
2. 结果格式：subagent 经常输出自然语言包装，比如“好的，以下是我的分析……”，聚合时很难程序化解析。
3. 并发副作用：多个 subagent 同时访问同一文件、API 或 MCP 工具时，容易出现争用、限流和重复写入。

## 做法 / 步骤

我目前比较稳定的做法是把主 agent 当作调度器，不让它直接处理业务内容。

**1. 拆任务，明确输入输出**

每个子任务必须满足三个条件：输入确定、输出 schema 确定、不依赖其他子任务的中间结果。例如批量分析仓库 README，任务可以拆成：

```json
{
  "task_id": "repo-001",
  "repo_path": "/data/repos/repo-001",
  "output_fields": ["summary", "language", "entrypoint", "risk"]
}
```

依赖关系要单独画出来，只把无依赖的分支并行。

**2. 给 subagent 固定模板**

每个 subagent 使用同一套 system prompt，明确角色、允许的工具、输出要求。输出要求直接写“只返回 JSON，不要解释”，比“尽量以 JSON 返回”有效。为每个 subagent 设置较低的最大轮数或 token 上限，防止失控。

**3. 有限并行派发**

不建议一次性拉起几十个 subagent。OpenClaw 里我会根据工具端承载能力，把并发限制在 3-5 个。伪代码骨架不依赖具体 SDK：

```python
tasks = load_tasks("tasks.jsonl")
results = {}

with ThreadPoolExecutor(max_workers=4) as pool:
    futures = {
        pool.submit(run_subagent, task): task["task_id"]
        for task in tasks
    }
    for future in as_completed(futures, timeout=180):
        task_id = futures[future]
        try:
            raw = future.result()
            results[task_id] = validate_schema(raw)
        except SchemaError:
            results[task_id] = retry_or_fallback(task_id)
        except TimeoutError:
            results[task_id] = {"status": "timeout", "task_id": task_id}
```

**4. 主 agent 只做校验与汇总**

subagent 返回后，主 agent 先做 schema 校验，再基于通过校验的结果生成最终结论。不要把所有原始输出拼接进上下文，否则主 agent 的上下文会迅速膨胀。

## 踩坑点

**输出漂移最普遍。** 即使 prompt 要求纯 JSON，部分 subagent 仍会加解释。解决方法是主 agent 侧做正则抽取或 JSON 解析，失败后自动重试一次，并把错误信息带回给 subagent。

**资源争用比想象中多。** 多个 subagent 写同一个文件或数据库表，容易互相覆盖。我会让每个 subagent 写独立临时文件，例如 `results/repo-001.json`，最后主 agent 统一读取合并。

**MCP 工具限流。** 如果多个 subagent 同时调用同一个 MCP 工具，可能触发服务端限流。控制并发数，并在重试逻辑里加入指数退避。

**隐藏依赖导致空转。** 有些任务表面独立，实际上需要前一个任务的结果。拆任务时如果不画依赖图，并行后会出现等待或错误。遇到这种情况，应该回退为串行或分批并行。

## 可复用建议

- 用任务清单文件维护状态，支持断点续跑。每个任务有 `pending / running / done / failed` 状态。
- 给每个 subagent 只传最小必要上下文，通过路径、ID 引用数据，不传大段原文。
- 设置全局超时和单任务超时，超时任务重试一次后转人工。
- 先跑 2-3 个任务验证 prompt 和 schema，再扩展到完整任务集。
- 低层工具要有并发保护，尤其是写操作，尽量做到幂等。

## 总结

subagent 并行编排的收益不是“让更多 AI 同时跑”，而是减少串行等待和上下文污染。它的成本主要在错误隔离、结果校验和并发控制上。对 OpenClaw 用户来说，先把任务拆到真正独立，再用固定 schema、有限并发和独立输出文件兜底，会比单纯增加 subagent 数量更可靠。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/95cf29963bc3eb91.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/f25b5f70e936803d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/05b6437830f93981.png)

