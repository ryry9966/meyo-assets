---
title: Agent 权限边界设计：在 OpenClaw 中实现分级确认
feedId: 32816
source: 综合讨论
publishedAt: 2026-08-12
---

## 一、背景：自动化失控的另一种形态

团队最近用 OpenClaw 搭了一个内部运维 Agent，能查日志、重启容器、修改开关配置。第一周运行很丝滑，直到某天凌晨三点，Agent 误读了一条告警，直接把生产环境的一个消费者组 scale down 到 0。没有确认，没有二次询问，几行 MCP 工具调用就完成了。

复盘时大家倾向于“加确认”，所有关键操作都要求人工批准。结果 Agent 退化成一个遥控器——每处理一条告警至少被中断两次，交互成本比手动操作还高。我们需要一套**工程化的权限边界**：让 Agent 在安全区内自主决策，在高风险区静默悬挂，绝不越过那条线。

## 二、问题定义：什么时候该问人

从实现角度看，“该不该问人”取决于三个维度：

- **可逆性与代价**：操作是否可回滚？失败代价多大？
- **上下文确定性**：当前决策是否依赖模糊的语义判断？Agent 对此有多高的置信度？
- **环境影响半径**：操作影响一个临时容器还是整个集群？

把这些维度映射成可执行的规则，是我们在 OpenClaw 中设计工具拦截器的核心目标。

## 三、做法：给工具打风险标签，引入分级确认

### 3.1 风险等级模型

我们定义四级风险标签，挂载到每个 MCP 工具或 Agent 插件上：

| 等级 | 标签 | 示例操作 | 默认行为 |
|------|------|----------|----------|
| **L0** | safe_read | 查询 Pod 列表、读取配置 | 自动执行 |
| **L1** | low_risk_write | 创建临时文件、打非生产标签 | 自动执行，记录日志 |
| **L2** | high_risk_mutate | 重启服务、修改开关配置 | **必须确认** |
| **L3** | destructive | 删除持久化资源、销毁实例 | **必确认+操作者双因素** |

这套分级的主要依据是可逆性：L0/L1 的操作在几十秒内可以安全回滚或影响极小；L2 有业务中断风险；L3 可能造成数据丢失。

### 3.2 在 OpenClaw 中落地

OpenClaw 的任务执行是基于 **Plan → Tool Invocation → Observation** 循环。我们在 Tool Invocation 前插入一个 `RiskInterceptor`，代码骨架如下（伪代码）：

```python
class RiskInterceptor:
    async def before_tool_call(self, tool_name, args, risk_level):
        if risk_level in (0, 1):
            return  # 放行
        # 风险≥2，发起人工确认
        ticket = await self.confirmation_channel.send(
            f"Agent 要执行 [{tool_name}]，参数: {args}，风险等级: L{risk_level}",
            timeout=120  # 秒
        )
        status = await self.confirmation_channel.wait(ticket)
        if status != "approved":
            raise ToolDeniedException("人工拒绝或超时")
        # 通过后日志记录审批流水
```

在实际部署中，OpenClaw 插件系统允许我们通过 **Tool Wrapper** 绑定风险等级。例如，一个 MCP 服务器暴露的 `restart_deployment` 工具可以在 `openclaw.yaml` 里配置：

```yaml
tools:
  - name: restart_deployment
    source: mcp://ops-server/restart
    risk_level: 2
    confirmation:
      required: true
      approvers: ["oncall","infra-admin"]
      timeout_seconds: 120
```

Agent 在规划阶段就能感知这些约束，不会把高风险操作和自主决策混排，从而在 Planning 输出中预留 Human-in-the-loop 步骤。

### 3.3 预授权窗口与智能放行

为了避免“凌晨三点叫醒人”的疲劳问题，我们对 L2 操作引入了**预授权窗口**：在变更窗口（如白天 10:00-16:00）内，同一类操作若已在一天内被同一审批人批准过，且参数相似（例如重启同一个 Deployment），Agent 可以获得 1 小时的自动放行票。这通过一个轻量的 Redis 缓存记录实现，并用 Key 校验。

## 四、踩坑记录

1. **确认通道不可靠导致 Agent 永久挂起**  
   最初用 Slack Bot 发消息等待按钮点击，但用户可能忽略或网络波动。我们必须设置 **硬性超时+默认拒绝**，超时后 Agent 记录“no_response”并推送告警到 PagerDuty，而不是无限等待。

2. **多次确认把阈值磨平**  
   如果一个任务需要连续执行 5 个 L2 操作，每次都确认会让流程断碎。我们改为**批量确认**：Agent 在执行前将同一任务组的所有 L2 工具调用打包成一个审批单，用户一次性批准整个操作集，然后 Agent 自主执行序列。

3. **上下文动态分级缺失**  
   起初所有环境统一标签，但 staging 环境的 delete 操作其实只是 L1。我们让 `openclaw.yaml` 支持环境上下文覆盖，`risk_level` 可以根据 `env: staging` 降级，降低确认成本。

## 五、可复用建议

- **在工具设计阶段就标注风险**，不要等 Agent 已经跑起来再补。MCP 服务器可以提供 `annotations` 字段，声明工具是否需要确认。
- **分级阈值可以调整，但永远保留 L3 硬防线**。破坏性操作无论何种环境都必须经过真人审批，并记录操作人。
- **确认通道要脱离 Agent 执行进程**，通过 webhook 或消息队列解耦，确保即使 Agent 重启也能恢复审批状态。
- **记录每次确认决策**，形成 Approval Log，方便事后审计 Agent 的自主行为是否越过边界。

## 六、总结

Agent 权限边界不是一条模糊的“重要操作问人”的线，而是可以通过风险分级、工具拦截、超时策略固化的工程方案。在 OpenClaw 这类允许深度定制执行引擎的框架下，我们可以做到：低风险场景快速自决，高风险操作主动悬挂，既避免“遥控器”困局，也杜绝“定时炸弹”式故障。当团队再次因为告警被叫醒时，至少不是去回滚 Agent 造成的破坏，而是去批准一个有上下文、可掌控的运维决策。

---

