---
title: 用 IDENTITY.md 给 OpenClaw Agent 装上可进化的灵魂
feedId: 31832
source: 综合讨论
publishedAt: 2026-08-06
---

## 背景：静态 System Prompt 的极限

在 OpenClaw 里跑长时间任务，或者让 Agent 在持续数周的项目里充当协作者时，一定会撞上同一个问题：最初的 system prompt 写得再好，场景一变就不够用了。比如：

- 项目初期定义的“严格的代码审查员”，第三轮迭代后团队希望它更关注测试覆盖率，但它依然机械地挑代码风格。
- Agent 在任务中学到了重要经验（例如某个 API 的鉴权坑），但重启后完全失忆。
- 换个用户、换个会话，角色设定的“成长”就归零。

本质原因是，system prompt 是静态的。想让 Agent 角色随着上下文进化，只靠给它塞更多消息是不够的。那相当于每次考试前重新看一遍从小学到现在的所有笔记——上下文窗口会撑爆，成本也不可控。

## IDENTITY.md 是什么

OpenClaw 生态里有一个鲜被讨论但工程价值很高的设计：`IDENTITY.md` —— 一个可以通过工具链读写的 Markdown 文件，专门承载 Agent 的角色定义、长期记忆和可演化的行为准则。它并不是官方强制的组件，而是社区在长期实践中沉淀下来的模式，类似把“给模型的心理画像”外化为一个可维护、可版本控制的文件。

它的核心思路是：

- 不再把角色描述和动态经验混在 prompt 里，而是抽出独立的 `IDENTITY.md`。
- Agent 可在运行期通过功能调用（function call）或插件读取这个文件，甚至有条件地更新它。
- 人类也可以随时用编辑器修改，作为对 Agent 行为的“安全介入点”。

## 问题：可进化的身份到底要解决什么

1. **长期任务中的角色漂移**——多轮对话或多次重启后，Agent 忘记了自己是谁，行为开始飘忽。
2. **经验无法沉淀**——一次任务中学到的教训，不能为后续任务复用。
3. **多实例协作冲突**——同一套身份被多个 Agent 实例共享时，如何让经验有控制地同步。
4. **提示膨胀**——把一切都塞进 system prompt，token 成本失控，且容易淹没关键指令。

## 具体做法：一个最小实现示例

以下是在 OpenClaw 中落地 `IDENTITY.md` 的一个可运行设计，只用内置工具和少量自定义逻辑。

### 1. 创建初始文件

在项目根目录或某个约定的 `identity/` 目录下创建 `IDENTITY.md`，结构可以这样：

```markdown
# Agent Identity

## Core Persona
- Role: Senior reliability engineer
- Tone: Direct, data-driven, favors simplicity
- Constraints: Never guess production configs; always request confirmation before destructive actions.

## Permanent Rules
- Prefer idempotent operations.
- Log all side effects.
- Escalate unknown unknowns to human.

## Evolving Memory
<!-- DYNAMIC: lessons learned during tasks will be appended here -->
```

固定部分用于约束核心行为，`DYNAMIC` 注释标记出 Agent 可以修改的区域。

### 2. 在 OpenClaw 中装配

利用 OpenClaw 的 `system` 配置加载文件内容。假定你使用 YAML 配置：

```yaml
system:
  template: |
    {identity}
    Current task context: {task_brief}
  variables:
    identity:
      file: identity/IDENTITY.md
```

如果用的是运行时 SDK，可以在启动时读文件后拼入 system prompt。

关键点：不要让整个 `IDENTITY.md` 作为一大段塞进每轮对话的 system prompt，而是让 Agent 在需要时通过一个自定义工具主动读取。这样可以节省 token 并减少“身份污染”。例如定义一个 `read_identity` 工具，返回文件内容；甚至提供一个 `update_identity_memory` 工具来写入新教训。

### 3. 让 Agent 能更新记忆

写一个安全的更新函数（以 Python 伪代码为例）：

```python
def update_identity_memory(new_lesson: str):
    path = "identity/IDENTITY.md"
    marker = "<!-- DYNAMIC:"
    with open(path, "r") as f:
        lines = f.readlines()
    insert_idx = next(i for i, l in enumerate(lines) if marker in l)
    lines.insert(insert_idx + 1, f"- {new_lesson}\n")
    # 可选：如果超过 N 行，触发压缩
    if len(lines) > 200:
        compress_dynamic_section(lines)
    with open(path, "w") as f:
        f.writelines(lines)
```

OpenClaw Agent 在识别到重要教训时调用该工具，如：

```
function_call: update_identity_memory(new_lesson="The /healthz endpoint on legacy-service requires header X-Forwarded-Proto to return 200.")
```

### 4. 定期压缩与去重

动态记忆无限增长会反噬。因此必须实现一个压缩函数（人工触发或按行数自动触发），将经验总结为更高层次的原则，去掉重复项。这可以交给语言模型本身完成：当动态区域超过阈值，就调起一个轻量总结任务，将现有经验提炼成“已学教训摘要”，替换掉原始条目，并在标题上加时间戳。

## 踩坑点

- **不可控的自我修改**——让 Agent 直接改写自己的核心行为规则非常危险。务必只允许它修改带有明确标记的动态区域，并且在关键决策路径上加入人类审批（例如通过推送 PR 的方式更新文件，而不是原地写入）。
- **多实例写入冲突**——如果多个进程共享同一个 `IDENTITY.md`，文件写操作可能损坏。用文件锁或者改为日志追加式（只 append 不重写），然后用定时任务合并日志到主文件。
- **上下文时效性**——Agent 读取 `IDENTITY.md` 后缓存在工作内存中，但文件可能已被外部更新。最好给 `read_identity` 工具加一个 `last_modified` 检查，发现过期就重读。
- **角色污染**——如果动态区域被大量任务无关信息填充，会导致 Agent 角色漂移。要区分“任务经验”和“角色设定”，经验最好按领域分块，并用分隔符标记，方便后续裁剪。

## 可复用建议

1. **版本管理**：把 `IDENTITY.md` 纳入 Git，每次 Agent 提议修改时自动开分支、生成 PR，人类审查后合并。这相当于给 Agent 的“心理成长”留下了审计记录。
2. **与记忆插件结合**：OpenClaw 生态有记忆向量库插件，`IDENTITY.md` 可以作为结构化元数据索引，指向更详细的经验片段，避免文件膨胀。
3. **分层身份**：对于复杂场景，可拆分为 `IDENTITY_CORE.md`（只读）和 `IDENTITY_MEMORY.md`（可写），前者存永久规则，后者存动态经验。
4. **实验性回滚**：保留定期快照。如果 Agent 行为因记忆更新而退化，可以快速回滚到上一个快照，对比差异定位问题。
5. **度量收敛**：监控 `IDENTITY.md` 文件大小和 Agent 在基准任务上的表现（如定义一组固定测试指令），确保进化是增益而非退化。

## 总结

`IDENTITY.md` 不是魔法，它是工程上把“模型角色”外化为可维护资产的一次务实尝试。在 OpenClaw 这样一个强调工具组合和工作流控制的 Agent 框架里，这种把身份文档化的思路，比不断在 prompt 上打补丁稳健得多。它让 Agent 的角色有了持续性、可审计性，以及一种受控的成长能力——而不是每次都从零开始教它做事。

如果你已经用 OpenClaw 跑了超过一周的长任务，不妨试一下把 prompt 里的角色描述抽出来放进一个可读写的 Markdown 文件。哪怕一开始只是手动更新，你也会发现维护 Agent 行为的一致性变得容易很多。

---

