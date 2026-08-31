---
title: AI Agent 的权限边界：什么时候该问人类，什么时候自己干
feedId: 35612
source: 综合讨论
publishedAt: 2026-09-01
---

## 背景：Agent 正在从“聊天”走向“动手”

当 Agent 只负责生成文本时，权限问题还不明显。一旦接入 MCP、插件或本地工具，它就有能力读文件、发请求、改配置，甚至执行 shell 命令。OpenClaw 这类项目让 Agent 通过工具调用和外部资源交互，但工具的执行往往缺少明确的权限语义。很多实践者要么让 Agent 事事都问，导致自动化体验稀碎；要么图省事全放权，直到某天 Agent 删错了目录、给所有人发了测试邮件，或者把半成品推到了生产。

核心问题不是“要不要给权限”，而是“什么时候给、给多大、怎么收回”。

## 问题：权限边界模糊带来的两类事故

一类是过度保守：Agent 每次写文件、改配置都弹确认，用户很快变成无脑点“允许”，确认形同虚设。另一类是过度激进：为了让自动化跑通，把高风险工具全部放行，一旦 Agent 产生误判或受到对抗性 prompt 影响，就可能造成不可逆损失。这两种情况都源于同一个缺陷：工具本身没有声明自己“有多危险”。

## 做法：用元数据声明工具风险，中间层强制拦截

比较务实的做法是维护一个 action registry，所有可被 Agent 调用的工具都需要声明自己是什么。至少包含三个字段：

- `risk_level`: read_only / low_risk / medium_risk / high_risk
- `requires_confirmation`: true / false
- `rollback_strategy`: none / reversible / manual

例如：

```yaml
- tool: list_files
  risk_level: read_only
  requires_confirmation: false
  rollback_strategy: none

- tool: send_email
  risk_level: medium_risk
  requires_confirmation: true
  rollback_strategy: manual

- tool: delete_directory
  risk_level: high_risk
  requires_confirmation: true
  rollback_strategy: none
```

在 Agent 执行工具前，中间层根据这份元数据做拦截。如果 `requires_confirmation: true`，就暂停执行，把工具名、参数、影响范围展示给用户，用户确认后再继续。对于 `read_only` 和 `low_risk`（比如写临时文件、记录日志），可以直接放行，但要保证可观测。

对于 MCP server，很多 server 暴露的工具并没有这种元数据。可以在本地包一层代理，或者利用 MCP 的 tool annotations 扩展。OpenClaw 的插件系统也可以采用类似的最小权限声明：插件安装时声明需要的 capabilities，运行时按声明的权限放行，超出声明范围的调用直接拒绝。

**分级示例**

- 自主执行：只读查询、写临时文件、生成草稿、格式转换。这些操作可逆或影响极小，适合全自动。
- 需要确认：发送消息/邮件、修改用户数据、git commit、创建云资源。这些是 low/medium risk，且有外部副作用。确认方式可以是弹窗、命令回显或一次性授权。
- 必须人工审批：删除生产资源、支付、批量修改、发布上线。这些高风险操作不仅要确认，最好有双人复核或 dry-run 预览。

## 踩坑点

1. **只读不等于无害**。Agent 读取敏感信息后可能通过另一个低风险通道外传。所以只读工具的放行，也要考虑输出是否可能泄露。对敏感数据源可以再加一层脱敏或审批。
2. **确认疲劳**。如果每个写操作都问，用户很快会无脑点“允许”。解决方法是引入“本次会话授权”“该工具本次任务不再询问”等临时授权，但要设置过期时间。
3. **权限元数据缺失时的默认值**。很多工具没有声明，如果默认允许，Agent 可能拿未声明的高风险工具乱用；如果默认拒绝，Agent 又寸步难行。建议默认拒绝，但提供快速补全元数据的流程。
4. **LLM 绕过权限描述**。不要只在 prompt 里告诉 Agent“不要做危险操作”，这很容易被忽略或诱导。要在工具调度层做硬拦截。
5. **沙箱不等于安全**。给 Agent 一个隔离环境，如果沙箱内有敏感挂载或网络通透，依然可能出事。检查挂载目录和网络策略。

## 可复用建议

- 把权限元数据作为工具接入的强制要求，没有声明就不给调度。
- 高风险操作提供 dry-run 模式，让 Agent 先输出“如果执行会发生什么”，人工确认后再真实执行。
- 所有工具调用记录审计日志，包括参数、结果、耗时、用户决策。出问题时能回溯。
- 对需要确认的操作，做成异步通知（如消息推送），不要让主流程长时间阻塞。
- 定期复盘权限配置：Agent 能力变化后，要重新评估哪些工具可以降权、哪些需要收紧。

## 总结

AI Agent 的权限边界不是一份静态配置，而是需要在“自动化效率”和“人为控制”之间持续调参。工程上，用元数据声明工具风险、用中间层强制拦截、用临时授权缓解确认疲劳，是一个可落地的起点。不要追求一次性把边界划得完美，先把高风险操作管住，再逐步放宽低风险操作，同时保留审计和回滚能力。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/168d398b7f0a0a07.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/2477011fd8eb54d2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-01/2f5ca94393a36496.png)

