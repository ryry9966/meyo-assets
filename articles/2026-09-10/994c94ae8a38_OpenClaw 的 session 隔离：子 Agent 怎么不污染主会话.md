---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 36826
source: 综合讨论
publishedAt: 2026-09-10
---

## 背景

在 OpenClaw 里跑多 Agent 任务，常见分工是：主会话负责拆解任务和做决策，子 Agent 去干重活——全库检索、长文档摘要、批量工具调用。理想状态下，主会话只需要拿到一句结论。

但实际跑久了，很多人的主会话会越来越"脏"：上下文持续膨胀、回复质量下降、token 成本线性上涨。排查下来，根因多半不是模型问题，而是子 Agent 的过程数据漏进了主会话。

## 三条典型的污染路径

1. **上下文共享**：子 Agent 直接复用主会话的 session，工具输出、中间推理全部留在同一条历史里。
2. **返回值失控**：子 Agent 本身隔离了，但 final result 把整段执行日志原样吐回来，等于换了个地方塞上下文。
3. **副作用残留**：子 Agent 往共享 memory 或共享工作目录里写文件、写笔记，主会话后续轮次被这些残留牵着走。

## 我们的做法：把 session 当进程管理

按 fork → exec → cleanup 的生命周期来处理：

**第一步，独立 session。** 子 Agent 用独立 session 启动，只注入任务描述和必要的前置上下文，不继承主会话历史。spawn 时显式传 parent_session 引用，子会话结束后引用即失效。

**第二步，定义返回契约。** 子 Agent 的最终输出强制走结构化 schema，例如 `{status, summary, artifacts[], token_used}`，summary 限制在 300 token 以内。超出部分只留文件路径，不留正文。

**第三步，工作区隔离。** 每个子 Agent 分配独立的 scratch 目录：主项目目录挂载为只读，临时区可写。产出的文件按约定放进 artifacts，主会话按需读取，不主动扫描。

**第四步，清理。** 任务结束即销毁子 session，工具缓存一并清掉，只保留 artifacts 和结构化结果。

## 踩坑点

- **相对路径错乱**：子 Agent 沿用了主会话的 cwd 假设，读错文件。子 Agent 的系统提示里必须显式声明自己的工作目录。
- **子 Agent"抢话"**：如果提示词带对话延续倾向，子 Agent 会试图向"用户"追问，导致任务挂起。子 Agent 提示要写成单轮任务式，明确"不做澄清，缺信息就按假设继续"。
- **MCP 工具结果回写主历史**：部分 MCP client 会把工具调用记录同步到主 session 的历史栈。要么在插件层关掉同步，要么给子 Agent 单独的 MCP 实例。
- **重试产生幽灵 session**：失败重试时没带幂等 ID，会攒下一堆残留会话悄悄消耗配额。每个任务发唯一 task_id，重试复用。

## 可复用建议

1. 返回契约越早定越好，schema 变更走版本号，避免新旧子 Agent 混跑。
2. 给子 session 单独开日志通道，排障时主日志不被噪音淹没。
3. 把"主会话单任务 token 增量"当作隔离效果的监控指标——持续超阈值，说明契约在泄漏。

## 总结

Session 隔离的本质是控制信息流向：允许子 Agent 自由制造中间过程，但回到主会话的只有结构化结论和文件指针。把 fork-exec-cleanup 做成标准模式后，我们主会话的单任务上下文占用降了一半以上，长任务的稳定性也明显改善。隔离不是锦上添花的性能优化，而是多 Agent 系统能不能跑得久的前提。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/c6c8e6300646632d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/7d28b066f8038c8a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/90f64776c88573f9.png)

