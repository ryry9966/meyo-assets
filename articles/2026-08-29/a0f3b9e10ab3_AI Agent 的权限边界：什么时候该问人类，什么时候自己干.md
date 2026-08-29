---
title: AI Agent 的权限边界：什么时候该问人类，什么时候自己干
feedId: 35245
source: 综合讨论
publishedAt: 2026-08-29
---

# 背景

Agent 接入 MCP、插件或本地工具后，能力范围会迅速从“聊天”扩展到读文件、执行命令、发请求、改配置、操作外部服务。此时真正的风险不是模型不够聪明，而是**权限边界没有按风险拆开**。

一类常见失败是太激进：Agent 自动执行了删除、覆盖、推送、发消息等不可逆动作。另一类是太保守：连查个状态都要确认，用户在弹窗里麻木点“允许”，Agent 反而不可用。两者都不是好设计。

# 问题

权限边界不能靠“感觉这里危险”或“我信这个 Agent”。需要把动作拆成可判断的风险对象，再决定默认策略。

关键点是：**危险通常不在工具名，而在参数和影响面**。`execute_command` 并不危险，`rm -rf /` 才危险；`write_file` 不一定危险，覆盖生产配置才危险。

# 做法/步骤

## 1. 给动作分级，而不是只给工具分级

可以按影响面和可逆性分四档：

- **R0 只读**：查询状态、看日志、读配置、列目录。
- **R1 可逆写**：新建文件、写临时目录、创建分支、发草稿。
- **R2 不可逆写**：删除、覆盖、强推、迁移数据、改生产配置。
- **R3 外部副作用**：发消息、调支付/短信、操作发布系统、触发真实人工流程。

## 2. 设默认策略

建议采用保守但可用的默认值：

| 级别 | 默认策略 | 附加条件 |
|---|---|---|
| R0 | 自动执行 | 写审计日志 |
| R1 | 自动执行或 dry-run | 限定路径/命名空间 |
| R2 | 必须确认 | 提供回滚方案 |
| R3 | 必须确认 | 预算上限、超时、熔断 |

## 3. 把“确认”做成结构化确认

不要只问“我可以继续吗？”这种开放式问题。应让 Agent 给出：

- 动作摘要
- 工具与参数
- 影响范围
- 是否可逆
- 回滚命令或恢复方式
- 意图 ID

结构化确认能降低用户判断成本，避免疲劳点击。

## 4. 在工具调用层落地

不建议只在 Prompt 里写“不要乱操作”。更可靠的方式是在工具执行前加一层策略包装，例如：

```ts
function runTool(tool, args, ctx) {
  const policy = matchPolicy(tool, args, ctx);

  if (policy.requiresConfirm) {
    const approval = askUser(structuredIntent(tool, args));
    if (!approval.approved) throw new AbortError();
  }

  const result = tool.run(args);
  audit.log(tool, args, policy, result, ctx);
  return result;
}
```

在 OpenClaw/Agent 这类框架里，可以给 MCP 工具注册元数据：`riskLevel`、`requiresConfirm`、`allowedParams`、`dryRunSupport`。执行前检查策略，执行后记录审计。

## 5. 支持自动批准规则

只靠人工确认不够，必须有可配置规则。规则应支持按参数匹配，例如：

- 允许 `git status`、`git log --oneline`
- 允许写 `/tmp/agent-test/` 下的文件
- 禁止任何包含 `rm -rf`、`DROP`、`TRUNCATE` 的动作

规则建议白名单优先、最小匹配，避免“一刀切”。

# 踩坑点

- **人类确认不等于安全**。弹窗过多会疲劳，用户会闭眼点“允许”。确认必须是低频、结构化、高风险的，否则形同虚设。
- **只控制工具名不控制参数**。允许 Python 执行但没限制 `subprocess`，等于没控制。
- **在错误层拦截**。MCP server 自身可能有权限，客户端策略拦不住 server 内部动作。要明确信任边界。
- **没有审计日志**。出问题后无法知道 Agent 是在哪个意图下执行、命中哪条规则。审计至少记录时间、工具、参数、风险级、决策结果、用户确认。
- **“只读”不一定无副作用**。某些命令可能锁表、占用资源、泄露敏感信息，需要单独评估。

# 可复用建议

1. **默认拒绝**，只放行明确允许的动作。
2. 开发环境可自动批准 R0/R1，生产环境 R1 也要确认。
3. 让 Agent 先提计划，再逐条批准。计划模式比单步确认效率高。
4. 所有敏感动作生成 intent record，落盘保留 N 天。
5. 定期检查自动批准规则，清理过宽规则。
6. 把权限策略和代码一起版本化，变更走 PR。

# 总结

Agent 的权限边界不是“信不信任 AI”，而是工程控制。问人类应该发生在高风险、不可逆、外部副作用的节点；其余可以自己干，但要有日志和可回滚设计。稳定版本不是“从不问”，也不是“什么都问”，而是能在正确的位置打断。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/91adce9679d296f9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/5ce06cf87e6cac9d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/c36bdfdd26a6afd9.png)

