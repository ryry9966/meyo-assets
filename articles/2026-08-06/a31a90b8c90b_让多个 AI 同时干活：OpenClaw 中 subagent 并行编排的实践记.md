---
title: 让多个 AI 同时干活：OpenClaw 中 subagent 并行编排的实践记录
feedId: 31846
source: 综合讨论
publishedAt: 2026-08-06
---

## 背景：为什么需要并行编排

在 OpenClaw 实际项目里，单个 Agent 串行处理多个独立任务时，等待时间会线性增长。比如需要同时爬取三个不同网站的结构化信息、分别调用代码执行工具验证数据、再汇总结果。如果让一个 Agent 按顺序来，不仅慢，还容易因为单任务的波动拖慢整体。

OpenClaw 的 subagent 机制天然适合“分而治之”：创建多个子智能体，每个挂载自己需要的 MCP 工具集，互不干扰地做事。但真的要让他们并行干活，而不只是“看起来并行”，就需要认真考虑编排、容错和资源控制了。本文记录了一次真实工程化实践，不依赖任何 SaaS 平台，全部基于 OpenClaw 的本地/自部署环境完成。

## 问题定义

目标：一个“主管 Agent”接收一条复合指令，将它拆成可并行的子任务，分发给多个 subagent 同时执行，最后统一收集结果、合并输出。

核心难点：
- 如何避免并发带来的 API rate limit 和本地资源耗尽？
- 某个 subagent 卡死或超时，怎样不影响其他任务？
- 共享有状态工具时，如何防止数据污染？
- 部分任务失败，还要不要汇总？怎么处理部分成功？

## 实现步骤

### 1. 拆解任务与定义 subagent 配置
使用 YAML 描述每个子任务对应的 subagent 定义（提示词、工具集、超时、重试）。例如：

```yaml
subagents:
  - name: "web_crawler_agent"
    tools: [ "mcp_web_fetch", "mcp_browser_screenshot" ]
    prompt: "你是一个网页信息提取专家，只提取指定字段。"
    timeout: 30
  - name: "code_exec_agent"
    tools: [ "mcp_python_repl" ]
    prompt: "你运行代码并返回输出，不做多余解释。"
    timeout: 20
```

这样就把每个 subagent 的角色和资源隔离出来。

### 2. 主管 Agent 的并行调度逻辑
主管 Agent 本身挂载一个“任务分发”工具，该工具用 Python 实现并行调度。伪代码结构：

```python
import asyncio
from openclaw import SubAgent

async def dispatch_and_run(task_items, config):
    semaphore = asyncio.Semaphore(5)   # 控制并发数
    tasks = []
    for item in task_items:
        tasks.append(managed_run(item, config, semaphore))
    # 设置全局超时，避免无限等待
    results = await asyncio.wait_for(asyncio.gather(*tasks, return_exceptions=True), timeout=120)
    return aggregate(results)

async def managed_run(item, config, sem):
    async with sem:
        sub = SubAgent(config["subagent_defs"][item.type])
        try:
            return await asyncio.wait_for(sub.run(item.prompt), timeout=config["timeout"])
        except asyncio.TimeoutError:
            return {"error": f"Task {item.id} timed out"}
        except Exception as e:
            return {"error": str(e)}
```

要点：
- 用 Semaphore 限制最大并行数，防止 API 限流或系统过载。
- 每个子任务用独立的超时包裹，避免某个 subagent 拖死整个 gather。
- 通过 `return_exceptions=True` 捕获异常，失败不中断其他任务。

### 3. 工具实例隔离
如果多个 subagent 共用一个 MCP 工具且该工具有内部状态（比如临时文件、会话上下文），很容易互相污染。实践里为每个 subagent 实例化独立的工具会话：在 `SubAgent` 初始化时传入独立的 tool instance，或利用 MCP 的 session 隔离机制，确保不会串。

### 4. 结果聚合与部分成功
`aggregate()` 函数会将正常结果和错误分开，生成一个结构化的汇总报告，包含成功率、每个任务的原始输出和错误信息。这样即使个别任务失败，也不会影响整体的可观测性。

## 踩坑点复盘

1. **并发数设置过于激进**  
刚开始取消了信号量，直接 `gather` 所有任务，结果遇到 MCP 后端服务的 rate limit 和文件描述符耗尽。引入 Semaphore 后，调优到 5～8 并发，吞吐反而更平稳。

2. **per-task timeout 忘记设置**  
一个 subagent 调用代码执行时进入了无限循环，因为只在 `gather` 外层设了全局超时，导致该任务占用线程不释放。改成每个子任务单独 `wait_for` 后，能精确释放资源。

3. **工具状态污染**  
某个 MCP 工具使用工作目录作为临时存储，并发生成文件时互相覆盖。解决：在 subagent 配置里为每个任务指定不同的 `TMP_DIR`，或者使用进程级隔离。

4. **结果合并时序非确定性**  
如果直接按完成顺序拼接结果，文档可读性差。改进：保留原始任务的 id，最后按 id 顺序排列，保证输出结构稳定。

## 可复用建议

- **封装 ConfigurableParallelAgent**  
将上述调度逻辑抽象成一个可复用的 Agent 类，通过 YAML 配置即可定义不同的并行工作流，避免重复写调度代码。

- **全链路日志**  
为每个 subagent 的执行打上 trace id，包含启动时间、耗时、工具调用链。出现异常时能快速定位是哪个子任务、哪一步 MCP 调用失败。

- **考虑幂等性**  
如果子任务有副作用（如发送邮件），需要设计 retry 策略，确保重试不会重复执行。可以在 subagent 指令中加“检查是否已执行”的逻辑，或使用幂等性工具。

- **渐进式启用**  
先在低风险场景使用（如信息收集、批量数据校验），验证稳定后再引入对时效性敏感的任务。

## 总结

在 OpenClaw 中用 subagent 做真正的并行任务编排，远比很多人想象得简单——核心就是异步调度 + 资源控制 + 错误隔离。但工程化落地时，必须正面处理并发限制、超时、工具隔离和部分失败。一旦把这几个点打磨好，就可以将原本数分钟的串行任务压缩到几十秒，显著提升自动化吞吐。这种模式尤其适合数据采集、多维度验证、并行代码执行等场景，值得在 OpenClaw 的日常实践中复现和迭代。

---

