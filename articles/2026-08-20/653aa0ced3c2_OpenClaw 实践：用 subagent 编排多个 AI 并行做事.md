---
title: OpenClaw 实践：用 subagent 编排多个 AI 并行做事
feedId: 33943
source: 综合讨论
publishedAt: 2026-08-20
---

## 背景

在 OpenClaw 上做自动化时，单 agent 通常既当指挥又当执行。任务少时没问题，一旦要批量抓取多个仓库、并行检查多个数据源、同时生成多份分析，问题就来了：上下文越塞越满，中间结果互相污染；任务串行执行，整体耗时线性增长；出了问题很难定位是规划错还是执行错。

后来我把长任务拆成“一个主控 + 多个 subagent”，让多个 AI 并行做事，效果稳定很多。这篇文章记录一下可复用的做法和踩坑点。

## 问题

1. **上下文窗口是硬约束**  
   把多个任务塞给一个 agent，它需要在不同任务的中间结果之间反复切换，容易丢失关键信息，也容易把 A 任务的结论误用到 B 任务。

2. **串行执行慢**  
   很多任务本来互相独立，但单 agent 会按顺序一个接一个跑，等 tool 返回、等网络请求，整体耗时被拉长。

3. **职责不清，排障困难**  
   规划、执行、校验混在一个 agent 里，一旦某一步出错，很难判断是任务拆解问题、工具问题还是上下文污染。

## 做法/步骤

### 1. 明确主控与 subagent 的边界

主控只做四件事：拆任务、派发、校验结果、汇总。  
subagent 只做一件事：执行单个任务，返回结构化结果。

例如批量检查多个 git 仓库，主控只负责生成任务列表，subagent 负责 clone、检查、返回 `result.json`。

### 2. 把 subagent 封装成 MCP tool 或插件命令

在 OpenClaw 里，不要在主 agent 的 prompt 里硬塞多个任务，而是把 subagent 调用封装成 MCP tool。这样主控可以用 tool call 的方式派发任务，每个 subagent 独立运行，互不干扰。

示意 tool 定义：

```json
{
  "name": "run_subagent",
  "description": "Run a single isolated subagent task",
  "parameters": {
    "task_spec": "string",
    "workdir": "string",
    "run_id": "string"
  }
}
```

关键点：每个 subagent 必须有独立 `workdir`，避免文件冲突；`run_id` 用于日志和追踪。

### 3. 并行派发

主控一次性生成多个 `run_subagent` 调用。如果 OpenClaw 插件层不支持原生并发，可以在一个 tool 内部用 shell 或 Python 实现并发。

例如用 `xargs -P 4` 启动多个进程：

```bash
echo "$task_ids" | xargs -P 4 -I {} sh -c 'run_subagent --run-id {}'
```

Python 侧可以用 `concurrent.futures.ThreadPoolExecutor`，并发数控制在 3-5。先串行跑通一个 subagent，再开并行，能减少排障成本。

### 4. 结构化汇总与校验

subagent 返回统一 JSON：

```json
{
  "status": "ok",
  "summary": "...",
  "files": ["result_a.md"],
  "errors": []
}
```

主控只读取 `summary` 和 `files`，不读取 subagent 的完整原始输出。校验 schema 后，如果发现 `status != ok`，只重跑失败项，不重跑全部。

### 5. 日志与追踪

给每个 subagent 分配 `run_id`，在独立目录写日志。主控汇总时引用日志路径，方便回看失败原因。

## 踩坑点

1. **并行不是越多越好**  
   受 API rate limit 和本机资源限制，并发 3-5 比较稳。开太多容易触发 429 或内存爆掉。

2. **上下文污染**  
   不要把所有 subagent 的原始输出都贴回主控。只回传摘要和文件路径，否则主控上下文很快耗尽，又回到单 agent 的老问题。

3. **文件锁冲突**  
   并行写同一文件、同一数据库会互相覆盖。每个 subagent 必须用独立目录，主控最后合并结果。

4. **假成功**  
   subagent 可能返回 `status: ok` 但内容为空。主控要做校验，比如检查 `summary` 非空、`files` 存在。

5. **超时与步数上限**  
   子任务没有上限时，一个 subagent 可能跑很久。建议给每个 subagent 设超时和最大步数，避免费用和时间失控。

## 可复用建议

- 固定 subagent 契约，统一 JSON schema，减少主控适配成本。
- 先串行跑通，再开并行。不要一上来就并发，否则排障很痛苦。
- 保留每个 subagent 的 raw output，不要只看摘要。摘要适合主控，raw 适合排障。
- 主控做成幂等，支持重试。任务失败后可以只重跑失败项。
- 把常用的 subagent 模板封装成 skill 或 plugin，例如 `fanout_search`、`fanout_code_check`，下次直接复用。

## 总结

Subagent 编排的本质，是把“一个长上下文串行 agent”变成“多个短上下文并行 worker + 一个轻量主控”。它并不复杂，但能显著改善上下文管理、执行速度和排障体验。

对于 OpenClaw 用户来说，最值得投入的是两件事：把 subagent 封装成 MCP tool，以及定义清晰的结构化返回契约。做好这两点，多数并行任务都能稳定落地。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-20/11c3329996cbf654.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-20/7a3f5f34e0bbcbf8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-20/9a613a0482c6f0f6.png)

