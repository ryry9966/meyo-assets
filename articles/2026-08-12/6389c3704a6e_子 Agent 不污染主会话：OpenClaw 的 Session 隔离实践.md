---
title: 子 Agent 不污染主会话：OpenClaw 的 Session 隔离实践
feedId: 32772
source: 综合讨论
publishedAt: 2026-08-12
---

## 背景：多 Agent 协作时一个隐蔽的上下文泄漏问题

在 OpenClaw 里搭建复杂自动化流程时，一个常见的模式是让“主 Agent”调度多个“子 Agent”去执行专项任务——例如代码审查子 Agent、文档生成子 Agent、外部工具编排子 Agent。这些子 Agent 通常会运行完整的推理链，包含中间思考、工具调用结果、甚至自我纠错过程。

如果没有做好隔离，子 Agent 在思考过程中产生的所有消息（包括内部试错、不成熟的结论）都会直接追加到主会话的消息历史里。主 Agent 看到这些“噪音”后，很容易被误导，或在不恰当的时机提前做出决策。更糟的是，在多轮对话场景下，上一个子任务的调试信息可能会泄露给最终用户，引发严重的隐私与合规风险。

这种问题在早期基于单链 Memory 的 Agent 框架中几乎无解，但 OpenClaw 提供了原生 session 隔离机制，让我们可以给子 Agent 开辟独立的上下文空间，只回传我们需要的结果。

## 问题复现：一行代码看清楚主会话污染

假设我们有一个 OpenClaw 主会话 `main_session`，它会在某个步骤调用一个子 Agent 去从数据库拉取报表数据并进行二次计算。

```python
# 伪代码：污染示例
sub_agent = Agent("data_analyst", model="gpt-4.1")

# 直接在主 sessions 上运行子 Agent
response = await sub_agent.run(
    "提取上个月销量，分析同比变化，输出 Markdown 报告",
    session=main_session   # 这里是问题根源
)
```

子 Agent 为了完成“分析同比变化”，可能会先尝试写出错误的 SQL 并修正，过程中会输出“不对，这个日期范围错了，需要加上时间戳转换”。这些中间推理都会进入 `main_session` 的消息列表。后续主 Agent 若基于历史消息做规划，就可能错误地认为数据已修正完毕，或者把内部纠错步骤当作最终结果展示给用户。

## 做法/步骤：利用 Session 隔离实现干净的子任务执行

OpenClaw 的 `Session` 对象可以独立创建，并与主会话完全分离。关键在于：**子 Agent 使用专用 session，只把最终结果注入主会话**。

具体步骤：

1. **为主流程创建 Session**  
   这个 session 会包含用户指令、主 Agent 的推理及所有需要暴露给用户的输出。

2. **为子 Agent 创建独立 Session**  
   不要复用主 session，而是调用 `openclaw.new_session()` 生成一个全新的上下文容器。

3. **在子 session 中运行子 Agent**  
   子 Agent 的所有思考痕迹、工具调用日志、中间消息都留在这个私有 session 里。

4. **提取最终结果并回传**  
   我们可以通过子 Agent 的返回值获取最终产出（通常是 `final_response` 或自定义结构），将该产出以一条用户消息或主 Agent 工具返回的形式追加到主 session。丢弃子 session 中其它所有消息。

下面是基于 OpenClaw 0.9+ API 的隔离实现示例：

```python
import openclaw as oc

# 初始化主会话
main_session = oc.new_session()

# 注册主 Agent 可调用的工具：隔离运行子 Agent
@oc.tool()
async def analyze_data(query: str) -> str:
    # 创建一个独立的 session，生命周期仅限此函数
    sub_session = oc.new_session()
    sub_agent = oc.Agent("data_analyst", model="gpt-4.1")

    try:
        result = await sub_agent.run(
            query,
            session=sub_session,
            # 推荐开启静默模式，降低噪声日志
            silent=True
        )
        # result.final_response 是子 Agent 的最终输出
        return result.final_response
    finally:
        # 可选：手动清理子 session（释放内存）
        await sub_session.delete()
```

主 Agent 只需像调用普通工具一样使用 `analyze_data`。子 Agent 的中间消息对主流程完全不可见，只有从函数返回的字符串会作为工具结果加入到主 session 的历史中。主 Agent 永远不会看到“不对，这个日期范围错了”这样的自我纠错。

## 踩坑点

实际工程中还会遇到几个细节问题，提前规避能节省大量排障时间。

**1. 子 session 中的系统提示容易遗漏**  
如果子 Agent 依赖特定的系统提示（如角色设定、输出格式强制），务必在创建子 session 时注入：

```python
sub_session = oc.new_session(
    initial_system="你是一个只输出合法 JSON 的数据分析师，不要输出任何解释。"
)
```

否则子 Agent 可能根据默认助手风格输出大量自然语言，污染返回结果的结构。

**2. 流式输出的处理**  
当使用 `stream=True` 时，子 Agent 的中间 token 会实时推送。如果我们在主流程中也采用流式，要确保只有主 Agent 的输出被推送给客户端。务必将子 Agent 的流式通道隔离——即子 Agent 的 `stream` 仅用于内部监控，不转发到用户侧。可以封装一个 `run_isolated_stream` 辅助函数，内部消费流、最后只返回完整文本。

**3. 工具调用结果与 token 消耗**  
子 session 会完整记录所有消息，包括工具调用结果。如果子 Agent 进行了大量搜索或文件操作，token 消耗会急剧上升。建议：

- 为子 Agent 设置较低的 `max_steps`，防止无限循环。
- 在提取结果后立即删除子 session，防止堆积。
- 监控子 session 的消息总数，超过阈值直接终止并返回错误。

**4. 并发子 Agent 时 session 的线程安全性**  
OpenClaw 的 session 对象不是线程安全的。如果主 Agent 需要并发执行多个子任务，每个子任务必须创建独立的会话实例，切不可共享。

## 可复用建议

基于多次生产环境踩坑，我总结了几个可以复用的模式：

- **封装一个 `isolated_invoke(agent, prompt, **kwargs)` 通用函数**  
  统一处理 session 创建、异常清理、结果提取。所有需要隔离的子任务都通过这个入口调用，避免散落的手动管理。

- **以工具形式暴露子 Agent 能力**  
  就像上面 `analyze_data` 那样，把子 Agent 包装成工具。主 Agent 通过 function calling 调度，子 Agent 的隔离对主 Agent 完全透明，架构更干净。

- **关键路径增加断言**  
  在主 session 中添加定期检查：如果发现主 session 历史中出现子 Agent 的典型调试话术（如“让我重新检查一下”），立即告警并中断流程。这个小型护栏可以在测试环境暴露未隔离的泄漏。

- **善用 OpenClaw 的 Session 元数据**  
  OpenClaw 允许给 session 附加标签（如 `session.metadata["role"] = "sub"`），结合日志过滤规则，可以快速绘制消息来源图，方便排查泄漏。

## 总结

子 Agent 不污染主会话的核心在于**物理隔离 session，仅回传语义级结果**。这不仅是为了主 Agent 的决策清晰，更是生产级多 Agent 系统必备的安全基础。OpenClaw 的 Session API 提供了足够灵活的上下文管理原语，我们再结合工程化封装，就能让主 Agent 始终拥有一个干净的推理空间——就像会议室里每个团队分别离席讨论，只回到主桌汇报最终结论。

希望本文的隔离模板和踩坑记录能帮助你在 OpenClaw 上构建更健壮的多 Agent 流程。

---

