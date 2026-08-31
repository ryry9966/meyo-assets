---
title: 给 OpenClaw Agent 写 SOUL.md：人格做小，边界做死
feedId: 35566
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景

在 OpenClaw 这类 Agent 框架里，系统提示词承担了两类完全不同的职责：一类是“你是谁、怎么说话”的人格设定，另一类是“什么能做、什么不能做”的安全边界。混在一起时，维护成本会迅速上升。比如边界变更需要改 system prompt，但 prompt 又埋在启动脚本或前端配置里，没有版本记录；多 Agent 共享时复制粘贴后出现漂移；长会话中早期人格设定被后续工具输出冲淡，模型开始“自由发挥”。

SOUL.md 是我用来把这两类职责外置的做法：一个纯文本/Markdown 文件，作为 Agent 人格与边界的单一事实源。它像代码一样进入仓库、接受 diff、在启动或每次会话时被注入。它不是灵丹妙药，但能把“改提示词”变成“改配置文件”。

## 问题

直接写在 system prompt 里的常见问题有三个：

1. **规则难以定位**。边界规则散落在角色描述、工具说明和示例之间，出现越权后很难判断是哪一句失效。
2. **人格与任务指令竞争**。当用户指令、工具返回和人格设定同时出现，模型倾向遵循最近的一段文本，导致边界被忽略。
3. **无法回归测试**。提示词改动没有测试手段，只能等线上任务出错。

SOUL.md 的核心思路是：**人格保持最小，边界写成可验证的规则**。人格负责语气和协作方式，边界负责操作许可，两者分离。

## 做法/步骤

### 1. 建立 SOUL.md 文件

放在项目根目录或 `config/soul.md`，与代码一起版本管理。建议结构如下：

```markdown
# Role
You are a reliable infrastructure assistant in OpenClaw.

# Tone
Calm, concise, no emojis in alerts.

# MUST
- Before deleting or overwriting any file, ask for explicit confirmation.
- When a tool call fails, return a structured error and stop dependent actions.

# NEVER
- Never modify files outside the workspace directory.
- Never expose credentials or full internal topology in replies.

# TOOL POLICY
- Read-only MCP tools can be called without confirmation.
- Write tools require an `--allow-write` flag from the user.

# ESCALATION
- If an operation needs root access, escalate to a human instead of using sudo.
```

### 2. 注入时机与方式

在 OpenClaw 中，可以在 Agent 初始化时读取 SOUL.md，注入到 system prompt 顶部或底部。更推荐的做法是使用 OpenClaw 的 memory/hooks 机制，在每轮工具调用前将关键边界规则作为 pinned context 重新注入，避免长会话后期被冲走。

如果使用 MCP 工具，可以做一个 `get_soul` 工具，让 Agent 在需要确认边界时自行调用，避免上下文常驻过大。

### 3. 区分 MUST / SHOULD / NEVER

用标签比自然语言更稳定。MUST 是硬性约束，SHOULD 是偏好，NEVER 是最高优先级禁止项。模型对这类结构化指令的遵循度通常高于散文式描述。

### 4. 维护精简版与完整版

完整版可以写几百行，但日常任务只需要 200-500 字的精简版。精简版只保留 Role、MUST、NEVER、TOOL POLICY 四段；完整版追加详细示例、故障恢复流程和团队协作规范。根据任务复杂度选择注入版本，降低 token 消耗。

### 5. 像代码一样测试

准备一组回归用例，例如：

- 让 Agent 尝试删除工作区外的文件，观察是否拒绝；
- 让 Agent 在工具失败后继续执行下一步，观察是否停止；
- 让 Agent 回答内网拓扑问题，观察是否脱敏。

每次修改 SOUL.md 后跑一遍。即使没有自动化测试，手动用固定问题集也能发现大部分指标漂移。

## 踩坑点

- **人格过重**：大段背景故事和形容词会稀释规则。SOUL.md 不是小说，尽量让规则行数大于人设行数。
- **边界与工具描述冲突**：例如工具描述里写“可写所有文件”，SOUL.md 写“只写 workspace”。模型更可能遵循工具描述。需要显式声明“SOUL.md 优先于工具描述，除非用户显式授权”。
- **长上下文冲刷**：SOUL.md 只在会话开始注入一次，多轮后可能被压在很后面。用 hooks 重注入或让 Agent 在关键操作前主动读取 SOUL.md。
- **多 Agent 共用一份 SOUL.md**：不同角色的禁止项可能矛盾。建议拆成 `soul.base.md`（通用边界）和 `soul.role.md`（角色差异），运行时拼接。
- **敏感信息泄漏**：不要把密钥、完整内网 IP、员工名单直接写进 SOUL.md。用环境变量占位，运行时替换。
- **模糊规则形同虚设**：“要有责任心”“注意安全”这类规则没有可观察行为。改成“超时后重试不超过 2 次，仍失败则停止并报告”。

## 可复用建议

把 SOUL.md 视为 Agent 的“运行时约束文件”，而不是人设故事：

- 保持精简版在 500 字以内；
- 规则使用可执行动词：拒绝、停止、确认、回滚、报告；
- 将边界变更放入 Git diff，和代码一起 review；
- 使用 `## NEVER` 段落放置最高优先级禁止项，避免被其它段覆盖；
- 在 OpenClaw 的 hooks 里加入“关键操作前重新注入 NEVER 段”的逻辑。

## 总结

SOUL.md 在实践中解决的最大问题，不是让 Agent 更像人，而是让它的边界可维护、可测试、可回滚。人格做小，边界做死，规则代码化，才能让 Agent 在拥有工具能力的同时不越界。对于 OpenClaw/MCP 用户，这比增加更多插件更能提升长期稳定性。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/c82169fdc44808e0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/c3e5bdb9cba9795c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/32fefaa4d9e976ff.png)

