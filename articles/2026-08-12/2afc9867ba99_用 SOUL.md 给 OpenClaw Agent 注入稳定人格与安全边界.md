---
title: 用 SOUL.md 给 OpenClaw Agent 注入稳定人格与安全边界
feedId: 32661
source: 综合讨论
publishedAt: 2026-08-12
---

## 背景：当 Agent 开始忘记自己是谁

在 OpenClaw 这类高度可组合的 Agent 框架里，我们通过 MCP 接入数据库、浏览器、本地文件系统，靠插件执行自动化任务，再把整个链条串联成一个可以“独立决策”的执行器。但实际跑久了就会发现一个尴尬的问题：Agent 的人格会漂移，边界会模糊。

比如你定义了一个“SRE 值班助手”，它的口吻应该冷静、克制，只允许使用预设的 MCP 服务器（比如 `ops-tools` 和 `incident-db`），不允许随意访问外网或执行 shell。但经过十几轮工具调用后，它可能在回复里突然变得啰嗦、卖萌，甚至尝试调用一个你没授权的 API。并不是 Agent 变聪明了，而是它基于长上下文的统计偏好把你最初塞在系统提示里的那几行指令淹没了。

SOUL.md 就是在这种背景下被社区逐步推出来的工程解法——把人格、语调、核心规则、工具白名单统统从代码或 prompt 模板中解耦出来，写进一个单独的、受版本控制的 Markdown/YAML 文件。它有点像虚拟角色的“出厂设定”，但更靠近配置即约束的运维思路。

## 核心问题：不止是“让回复更拟人”

很多开发者把 SOUL.md 当成一个写角色小作文的地方，这其实是本末倒置。在工程化场景里，SOUL.md 真正要解决的是三个问题：

1. **跨会话的持续性偏差**  
   即使你在每次请求时都拼接系统指令，一旦上下文变长、工具返回的结果结构复杂，Agent 对自身身份的坚守就会衰减。SOUL.md 需要成为第一条未被稀释的 anchor。

2. **边界逾越的廉价成本**  
   当一个 Agent 接入 5 个 MCP 服务器、10 个插件时，没有集中声明“你能用什么、绝对不能碰什么”，任何一次模糊匹配的推理都可能变成一次危险的误操作。

3. **多实例维护的一致性灾难**  
   如果你在代码里硬编码人格，每改一个字都要重新部署、重新测试。SOUL.md 作为外部配置文件使得人格迭代可以独立于 Agent 主逻辑。

## 做法：OpenClaw 下的 SOUL.md 设计范式

下面是一种经过验证的、可以直接复用的结构。我们在项目根目录放置 `soul.md`，YAML front matter 负责机器可读的元数据，Markdown 区块负责给 LLM 阅读理解。

```yaml
---
name: SRE NightOps
version: 1.2.0
model_min_context: 8000
permissions:
  allow_mcp:
    - ops-tools
    - incident-db
  deny_plugins:
    - browser
    - code_exec
---
```

Markdown 部分则使用固定标题层级，确保每次注入系统提示时格式稳定：

- **Who you are**  
  身份陈述：例如“你是某公司基础设施团队的值班 Agent，具备 5 年 SRE 经验，沟通风格冷静、直接，只基于日志和监控数据给出建议”。

- **How you reply**  
  语气范例：限制回复长度（单条不超过 300 个 token），禁止使用 Emoji，明确禁止臆测不存在的告警。

- **Absolute rules**  
  这里必须用绝对化语言。不要写“尽量避免调用外部服务”，而要写“**永远不要**调用未被 `permissions.allow_mcp` 列出的任何工具；**永远不要**在回复中编造日志行”。

- **Permanent memory**  
  持久化记忆片段：比如“当前值班环境为 AWS us-east-1，登录用户是 `opsbot`”。

在 OpenClaw 中加载时，我们会在 Agent 初始化文件的最前端拼接：

```python
import yaml
from pathlib import Path

def load_soul(agent_name: str) -> str:
    soul_path = Path(f"configs/{agent_name}/soul.md")
    with open(soul_path) as f:
        content = f.read()
    # 简单校验 front matter 完整性
    if not content.startswith("---"):
        raise ValueError("Invalid SOUL.md: missing front matter")
    return content

system_prompt = load_soul("sre-nightops") + "\n\n" + runtime_rules
```

这样做的好处是，即使后续运行时规则有变化，SOUL 部分的优先级始终最高。

## 我踩过的坑

**1. SOUL.md 膨胀到 3000+ tokens**  
最初我把大量历史错误案例塞进 Permanent memory，导致每次请求凭空多出半个上下文窗口。后来拆成“核心人格（soul.md）+ 案例库（向量检索）”，按需注入近邻案例，才把大小压回 900 tokens 以内。

**2. 温柔的边界等于没有边界**  
一条规则写成“尽量不要使用未经授权的工具”，Agent 在实际推理中有相当大概率忽略。改成“如果工具不在 allowed_mcp 列表中，立即返回错误并停止动作”，才有效阻断。边界必须可执行，不能是愿望清单。

**3. 动态更新后旧人格缓存残留**  
有段时间我们用内存缓存了 SOUL.md 的内容，结果运维修改了边界策略，Agent 却在重启前一直用旧版。现在每次对话启动都重新读取文件，副作用是 I/O 频率略增，但一致性保证了好几个量级。

**4. 与 MCP 工具描述相矛盾**  
SOUL.md 里声明“禁止发送邮件”，但 MCP 服务器 `comms` 的工具描述里写的是“send_email: 发送邮件给任何人”。Agent 在困惑时有时会优先相信工具描述。解决办法是让 SOUL.md 中的 `deny` 列表在代码层面直接过滤工具可用性，而不只是文本约束。

## 可复用建议

- **从“角色卡模板”起步**  
  先在团队内沉淀 3～5 套稳定模板（SRE 助手、数据分析师、客户支持 Agent），随后新项目直接基于模板裁剪，避免每人重写一遍。

- **用测试守护人格**  
  写几条 pytest，输入包含越界请求，检查 Agent 是否按 SOUL.md 给出了明确拒绝，并返回预设的错误码。这种回归测试比人眼审查可靠得多。

- **把 SOUL.md 版本号打入日志**  
  每次 Agent 启动时记录 `soul_version`，出问题时你能立刻回溯到是边界定义变了，还是 LLM 推理跑偏。

- **保持 YAML front matter 与 Markdown 语义同步**  
  不要在 YAML 中声明 `allow_mcp: [ops-tools]`，然后在 Markdown 里又写一遍“只能使用 ops-tools”。维护单一事实源，否则双重声明终有一处会过时。

## 总结

SOUL.md 并不是要取代系统提示，而是为 Agent 提供一份不随对话长度稀释、不被工具描述冲淡、不被多实例部署搞乱的“人格与边界基线”。对于 OpenClaw 这种把自主决策权放得很开的框架，提前把边界固化下来，是最低成本的避坑手段。建议大家在下一个正在维护的 Agent 项目里尝试引入，先从小处做起——哪怕只定义清楚“你是谁，绝对不能做什么”，你都会发现后续排障和信任建立顺畅得多。

---

