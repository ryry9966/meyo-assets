---
title: Subagent 编排实战：让多个 Agent 并行做事，而不是排队聊天
feedId: 35593
source: 综合讨论
publishedAt: 2026-09-01
---

## 背景

在 OpenClaw 这类 Agent 自动化场景里，单主 Agent 很容易变成“串行办事员”：先读文件，再调 MCP，再搜索，再总结。每个步骤看起来都不慢，但一多起来延迟线性叠加。尤其是批处理类任务——比如同时检查多个仓库、拉取多个 API、审查一批配置——天然适合并行。

这时候就需要 subagent：主 Agent 只负责拆任务、派发、汇总结果，具体脏活让多个子 Agent 各自在独立上下文里完成。

## 问题

直接让主 Agent 自己并行调工具，会出现几个典型问题：

- 多个任务的中间结果混在一个上下文里，越跑越乱；
- 一个任务失败或超时，容易影响整体判断；
- 主 Agent 读太多细节，并行收益被上下文膨胀吃掉；
- 输出格式不统一，汇总时只能靠“再总结一次”，误差变大。

所以更好的方式不是“主 Agent 多线程聊天”，而是把 subagent 当作有明确契约的工具。

## 做法/步骤

### 1. 先判断能不能并行

只拆无共享写、只读或独立写、可幂等执行的任务。比如“检查 10 个仓库的 README 是否规范”就适合；“先改 A 文件再根据 A 结果改 B 文件”就不适合。

### 2. 给每个 subagent 定义契约

建议每个 subagent 只做一个窄任务，并且输入输出结构化。示例：

```yaml
name: repo-audit
input:
  repo_url: string
  checks: string[]
output:
  ok: boolean
  findings: string[]
  raw_log: string
tools: [git_read, file_read, grep]
max_tokens: 2000
timeout_s: 120
retry: 1
```

在 OpenClaw/MCP 里可以把这种 subagent 封装成一个标准 tool，带上 `inputSchema` 和 `outputSchema`。这样主 Agent 调用它和调用普通 MCP 工具没有区别。

### 3. 主 Agent 只派发和汇总

主 Agent 不要逐个阅读 subagent 的完整日志，只需要收集结构化结果。并发调用时建议限制并发数，避免同时开太多导致 token 成本或本地资源冲高。

```python
async def dispatch(subagent, tasks, limit=6):
    sem = asyncio.Semaphore(limit)

    async def run(t):
        async with sem:
            return await subagent.run(t)

    results = await asyncio.gather(
        *[run(t) for t in tasks],
        return_exceptions=True
    )
    return [normalize(r) for r in results]
```

### 4. 汇总时做严格校验

每个 subagent 返回后，主 Agent 先校验 schema 和 `ok` 字段。超时或失败的任务先重试一次，如果还不行就标记为 `partial`，不要把失败项静默丢掉，更不要让主 Agent 脑补结果。

## 踩坑点

- **上下文膨胀**：subagent 输出太啰嗦。限制输出为 JSON，并明确要求“只返回结构，不解释过程”。
- **共享文件冲突**：多个 subagent 同时写同一个文件会互相覆盖。让每个 subagent 写独立目录，最后由主 Agent 合并。
- **超时叠加**：并行任务的总超时不是简单相加。给每个子任务单独 timeout，同时设置整体 deadline，避免一个慢任务拖住全局。
- **重试风暴**：失败就整体重试可能产生重复副作用。只对只读或幂等任务做自动重试。
- **主 Agent 过度参与**：如果主 Agent 把每个 subagent 的细节都读一遍，并行的收益就被吃掉了。让 subagent 拥有局部上下文，主 Agent 只看结论。

## 可复用建议

- 统一返回结构：`{ "ok": true, "error": null, "data": {} }`。
- 每次运行给 subagent 生成 `run_id`，记录输入 hash、输出 hash、耗时，方便排障。
- 先拿 2～3 个任务跑小样本，验证 schema、成本和并发限制，再扩大规模。
- 把 subagent 当工具，不要让它做计划。职责单一，越窄越稳。
- 在 OpenClaw 中优先把 subagent 封装成 MCP tool 或独立 worker，避免主 Agent 直接展开所有指令。

## 总结

Subagent 编排解决的是延迟和上下文隔离问题，不是万能提效药。关键在三点：任务可并行、契约清晰、失败可降级。只要把派发、执行、汇总三层分开，多个 Agent 并行做事才真正可靠，而不是从“排队聊天”变成“同时聊天”。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/cb43db6a5f0a4460.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/f699744a9150ee3c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/436b5d79bb8498ef.png)

