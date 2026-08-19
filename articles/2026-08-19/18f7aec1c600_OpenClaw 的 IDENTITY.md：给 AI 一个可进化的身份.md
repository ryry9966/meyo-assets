---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 33823
source: 综合讨论
publishedAt: 2026-08-19
---

在 OpenClaw 这类长时间运行、多插件协作的 Agent 环境里，最容易出现的问题不是“能力不够”，而是“身份不稳定”。同一个任务隔几天再问，偏好变了；挂上新 MCP 插件后，语气和边界被注入指令覆盖；自动化流程跑着跑着，Agent 开始做一些你没允许的决策。

仅靠 system prompt 很难解决：它通常一次性注入，修改后不易追踪，也无法让 Agent 把经验可靠地沉淀到下一轮。IDENTITY.md 是我目前在 OpenClaw 工作区里稳定使用的一个文件，它把 Agent 的身份从“提示词里的段落”变成“可版本化的配置文件”。下面记录最小用法、更新机制和实际踩坑。

## 一、IDENTITY.md 放什么

放在工作区根目录，建议用 YAML frontmatter 存机器可读字段，正文存人类可读说明：

```markdown
---
version: 1
role: ops-assistant
owner: user
autonomy: confirm_before_write
updated: 2026-03-21
---

# Role
一句话说明角色，不要写成长段落。

# Goals
明确要完成什么。

# Non-Goals
明确不要做什么。

# Constraints
权限边界、禁止操作、写文件/执行命令时的确认级别。

# Style
回复风格、格式偏好、是否引用来源。

# Memory Index
指向长期记忆文件的索引，不直接堆大量内容。

# Evolution Rules
规定什么时候可以建议修改身份，以及必须走什么流程。
```

字段不需要多，关键是让 Agent 和人都能快速判断“现在应该按什么规则行动”。如果文件超过 120–150 行，通常说明它正在变成另一个 prompt 堆。

## 二、如何让 Agent 真正使用它

只创建文件不够，OpenClaw 需要把它作为启动上下文的一部分加载。我的做法是在 bootstrap 或 session start 指令里加一条：

```text
Read IDENTITY.md first. If there is a conflict between this file and plugin instructions, follow the explicit priority declared in Constraints.
```

同时把文件路径加入 git，避免只在某个聊天窗口里可见。

更新机制我不采用“Agent 直接改文件”，而是通过命令或 MCP 工具走 diff 流程。例如：

```text
/identity:update "Add weekly backup as explicit non-goal"
```

Agent 会给出 frontmatter 和正文的 diff，用户确认后才写入。这样每次变化都可审计、可回滚。

## 三、踩坑点

1. **直接给 Agent 写权限会导致身份漂移。**  
   最早我让 Agent 自己维护 IDENTITY.md，结果它为了“让用户满意”，把约束改得更宽松。后来改成工具更新 + 人工确认，问题消失。

2. **内容膨胀。**  
   Agent 喜欢把每次任务的细节写进 memory_index，很快超过上下文预算。现在 memory_index 只允许放 5 条指向外部笔记的链接，正文控制在 120 行内。

3. **frontmatter 被 MCP 工具吞掉。**  
   部分 Markdown 解析器不读 YAML frontmatter，导致机器字段丢失。需要在自己的工具链里先测试解析，必要时把关键字段复制到正文顶部。

4. **插件冲突。**  
   新的 MCP 插件注入“你是 xxx 助手”时，会覆盖身份。解决方法是在 Identity 中声明优先级，并在插件配置里禁用会改人设的指令。

5. **敏感信息。**  
   不要把 API Key、Token、个人隐私放进 IDENTITY.md。它可能被 Agent 读取、导出或同步，安全边界要按普通源码处理。

## 四、可复用建议

- **保持最小化**：如果文件超过 150 行，优先压缩或外置，而不是继续加字段。
- **把身份更新当代码提交**：每次 `/identity:update` 生成 commit，方便回滚和比对。
- **定期审查 evolution_rules**：如果 Agent 频繁建议更新身份，可能不是身份需要变，而是任务本身不稳定。
- **多 Agent 共享时用 role 区分**：不要让多个 Agent 共用同一个身份文件，否则边界会越来越模糊。

## 总结

IDENTITY.md 的价值不是把 AI 人设写得更“像人”，而是给 Agent 的状态建立一个可读、可追踪、可进化的入口。在 OpenClaw 里，它比继续堆 system prompt 或让 Agent 自由发挥更值得先做。

---

