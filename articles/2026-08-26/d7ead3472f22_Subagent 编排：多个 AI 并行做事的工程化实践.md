---
title: Subagent 编排：多个 AI 并行做事的工程化实践
feedId: 34822
source: 综合讨论
publishedAt: 2026-08-26
---

## 背景

单个 Agent 处理长任务时，通常会遇到三类问题：上下文窗口被中间结果占满；工具调用链过长导致错误累积；任务串行执行慢，且失败后很难定位是哪一步出了问题。

Subagent 编排是把一个大任务拆成多个边界清晰的子任务，派发给多个子 Agent 并行执行，再由主 Agent 汇总。这个思路在 OpenClaw 这类可扩展 Agent 环境中尤其适合：主 Agent 不再亲自做所有事，而是负责拆解、派发、验收和合并。

## 什么时候适合用 subagent

适合的场景：

- 任务可以拆成彼此独立、输出边界清晰的子任务；
- 单个子任务上下文很重，或者需要不同工具集；
- 需要并行提速，例如多源检索、多文件生成、独立测试。

不适合的场景：

- 子任务强依赖、需要实时共享状态；
- 任务太小，调度成本高于执行成本；
- 副作用强相关，例如多个 Agent 同时改同一个文件。

## 做法步骤

### 1. 先定义边界

每个 subagent 只做一件事，输入输出要有明确 schema。例如在代码改造任务中，可以拆成 `repo-researcher`、`code-writer`、`reviewer` 三个角色。不要把所有能力都塞给一个 subagent，否则又回到了单 Agent 的问题。

### 2. 配置隔离

给每个 subagent 独立配置 system prompt、工具白名单、模型和最大步数。写操作默认关闭，只给读取类工具；确实需要写文件或调用外部服务时，通过 MCP 工具做受控写入，并且尽量先 dry-run。

### 3. 并行调度

没有依赖的子任务同时发起；有依赖的按 DAG 分层执行。不要无脑全部并行，建议设置 `max_concurrency`，例如同时只跑 3 个 subagent。这样既能提速，又不会瞬间打满 API 限流。

### 4. 验收和汇总

强制 subagent 返回结构化 JSON，例如：

```json
{
  "task_id": "repo-researcher",
  "status": "done",
  "artifacts": ["docs/api.md", "src/search.ts"],
  "summary": "...",
  "confidence": 0.9
}
```

主 Agent 只汇总 artifacts、冲突点和 status，不重做内容。必要时再引入一个 `reviewer` subagent 检查不同子任务之间的冲突。

### 5. 失败处理

每个子任务都要设置超时、重试次数和降级策略。例如检索失败时返回空列表并标记 `status: skipped`，不要让主 Agent 无限等待。

## 踩坑点

- **限流与账单**：并行一开，API 请求成倍增加。建议本地排队、限制并发，短任务用更便宜的模型。
- **输出漂移**：即使约束 JSON，模型也可能加 Markdown 围栏或解释文字。解析时先剥离代码块，再做容错；或者在 prompt 里写“只输出 JSON，不要解释”。
- **上下文断裂**：只返回摘要会让主 Agent 缺失关键细节。要求 subagent 返回文件路径、行号、引用片段，而不是复制大段文本。
- **副作用冲突**：多个 subagent 同时写同一路径会互相覆盖。按目录或文件分片，或者加锁。
- **权限失控**：subagent 继承主 Agent 全部工具时可能误删文件。必须做工具白名单，写操作需要审批。
- **假完成**：主 Agent 汇总时可能把未完成项写成已完成。要求 status 明确，并且每个完成项都要有 artifact 支撑。

## 可复用建议

- 从 2~3 个 subagent 起步，任务单一；
- 用编排文件描述 subagent 拓扑：role、input、output_schema、tools、timeout、retry；
- 给每个 subagent 写明确的停止条件，避免无限探索；
- 所有 subagent 输出 trace log，方便回溯问题；
- 准备固定 eval 集，检查并行结果是否一致；
- 用 MCP 工具封装副作用操作，权限容易收口。

## 总结

Subagent 编排的核心不是“让多个 AI 同时干活”，而是拆分边界、隔离上下文、控制副作用、验证汇总。先做并行只读任务，再逐步放开写权限，比一步到位要稳得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/fd90d27ef708833c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/889b8cba9f73392d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/a511c730e792f968.png)

