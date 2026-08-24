---
title: 给 OpenClaw 一个可进化的身份：IDENTITY.md 工程实践
feedId: 34597
source: 综合讨论
publishedAt: 2026-08-25
---

## 背景

OpenClaw 的 Agent 在长任务、多会话或自动化流程里很容易出现“身份漂移”：前一个会话还能守住边界，后一个会话就开始越权调用工具、改变语气或遗忘核心职责。把身份全部塞进 system prompt 也不够理想，因为每次调整都要改模板、重启实例，多人协作时更会版本混乱。

IDENTITY.md 提供了一个持久化、可版本化、可审计的身份文件。它让 Agent 有一个稳定的“自我定义”，也可以在受控条件下演进。

## 问题

直接维护长 system prompt 会带来几个工程问题：

- 职责边界模糊，改一处要同步多个 prompt 模板。
- Agent 在任务中“学到”的经验无法安全固化。
- 团队协作缺少唯一身份来源，不同环境行为不一致。
- 身份信息与 MCP 工具、插件权限脱节，Agent 会自信地调用不存在的能力。

## 做法 / 步骤

### 1. 建立最小身份模板

IDENTITY.md 只保留最关键的身份信息，不要把操作手册、FAQ 都塞进去。建议包含：

```markdown
# Agent Identity

- 角色 / 职责
- 输入输出范围
- 语气与风格
- 可用工具
- 禁止事项
- 记忆更新规则
```

控制在 80～150 行以内，超过后模型注意力会被稀释。

### 2. 让 IDENTITY.md 成为唯一身份源

在 OpenClaw 启动或会话创建时加载 IDENTITY.md。其他 prompt、插件说明、MCP 工具描述只引用它，不重复定义。例如：

> 工具权限以 IDENTITY.md 中的 “可用工具” 列表为准，MCP server 只负责提供能力。

### 3. 设计受控的自更新机制

不要让 Agent 在任务结束后直接覆写 IDENTITY.md。更稳妥的做法是：

1. Agent 输出“身份变更建议”：说明改哪一段、理由、影响范围。
2. 由人工或 review 流程合并到 IDENTITY.md。
3. 保留变更记录，方便回滚。

这能避免一次失败任务污染长期身份。

### 4. 版本化管理

把 IDENTITY.md 纳入 Git。commit message 写清楚触发变更的任务和决策背景。这样当 Agent 行为异常时，可以快速定位是身份改坏了，还是工具配置变了。

### 5. 与 MCP / 插件权限对齐

IDENTITY 中声明的“我能调用 X”必须与实际 MCP server 配置一致。建议在启动时做一次一致性校验：发现 IDENTITY 中列出但未配置的工具，直接告警或拒绝启动，避免 Agent 反复尝试不存在的能力。

## 踩坑点

- **自动写入污染身份**  
  某次失败任务可能让 Agent 把“暂时不要做某类操作”写成“永远不做”，导致能力退化。所以身份更新必须经过人工或 review，不要 auto-commit。

- **身份文件过长**  
  很多人会把具体任务步骤、脚本说明也写进去。身份文件超过 150 行后，关键约束容易被忽略。操作细节应放到 docs 或工具描述里。

- **权限脱节**  
  只改 IDENTITY 不更新 MCP 配置，或者反过来。Agent 会“一本正经”地给出错误方案。启动时校验可以避免这个问题。

- **多 Agent 共用一份文件**  
  多个 Agent 共用一个 IDENTITY.md 时，容易互相覆盖。建议每个 Agent 保留独立文件，公共约束用引用或 include 方式共享。

- **只有正向描述**  
  只写“我应该做什么”，缺少“不做什么”和“何时升级给人”。负向约束与升级路径同样重要。

## 可复用建议

- 用 frontmatter 结构化元数据：`version`、`owner`、`updated_at`、`scope`、`tools` 等。
- 身份变更走 PR / MR 流程，至少一人 review。
- 每两周检查一次 IDENTITY 是否与真实任务一致，移除过期条目。
- 保存旧版本，保证可回滚。行为异常时先回滚身份再排查其他配置。
- 明确升级路径：当请求超出边界、工具连续失败或出现不确定场景时，Agent 应停止并请求人类确认。

## 总结

IDENTITY.md 不是一份静态 system prompt，而是 Agent 的长期契约与可进化的记忆。把它当作代码一样管理：版本化、审阅、回滚、对齐权限。只有这样，OpenClaw 中的 Agent 才能在多会话、多插件和自动化流程中保持稳定且可预期。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/ec7858148628a1c4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/d1825242d6ce7c33.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/8f3d8d10844db248.png)

