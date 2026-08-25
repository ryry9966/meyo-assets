---
title: AI Agent 权限边界：什么时候该问人类，什么时候自己干
feedId: 34645
source: 综合讨论
publishedAt: 2026-08-25
---

## 背景

在 OpenClaw 这类 Agent 框架里，模型不是只会聊天，它通过 MCP、插件、shell 工具接入了文件系统、浏览器、消息接口、CI/CD 等真实世界。权限边界决定它是一次可靠自动化，还是一个事故源。

模型不会天然理解“删除生产数据”比“读一个网页”严重。它看到的只是工具描述和参数。所以权限边界必须在配置层做，不能靠模型临场判断。

## 问题

常见有两种极端：要么所有写操作都弹确认，Agent 每步都被打断；要么为了顺畅直接 `allow: ["*"]`，然后让一个看似无害的清理任务变成删库。

真正的问题不是“该不该信任 Agent”，而是“当它判断错误时，系统还能不能兜住”。

## 做法/步骤

**1. 给工具分级，而不是给模型分级。**  
用“副作用”和“可逆性”两个维度划分：

- L0 只读：浏览网页、查询文件、获取状态
- L1 本地可逆写：新建文件、写草稿、生成临时目录
- L2 外部可撤回：发消息、创建 issue、提交评论
- L3 不可逆/生产影响：删除资源、合并分支、发布版本
- L4 高危：支付、提权、修改权限策略

**2. 配置三层策略：allow / ask / deny，别用布尔白名单。**  
示例只是表达分层思路，不一定要照抄：

```yaml
permissions:
  fs.read:    { effect: low,      policy: allow }
  fs.write:   { effect: medium,   policy: ask, scope: "./workspace/**" }
  shell.exec: { effect: high,     policy: deny, allowlist: ["git status", "ls -la"] }
  billing.pay:{ effect: critical, policy: deny }
```

**3. 审批流要有上下文。**  
当 Agent 请求写操作时，不要只问“是否允许？”。要展示：工具名、完整参数、目标资源、副作用等级、是否可回滚、审批超时后的默认动作。默认超时应为拒绝，而不是自动放行。

**4. 命令类工具先 dry-run。**  
如果 MCP shell 工具不支持 dry-run，就在沙箱或只读容器里先跑一遍，确认参数展开、路径、目标都符合预期。尤其对 `rm`、`mv`、`sed -i`、`git push --force` 这类命令。

**5. 记录决策上下文，不只是操作日志。**  
至少包含：用户原始指令、模型选择的工具、参数、权限策略版本、审批人、审批时间、是否带 scope。这样出问题后可以还原“它为什么这么干”，而不是只看到“它执行了删除”。

## 踩坑点

- **白名单过宽**：`allow: ["*"]` 等于没有边界。我见过一次清理临时目录的任务，模型执行 `rm -rf ./tmp`，但工作目录被设置成项目根目录，结果删了整个仓库。白名单里如果包含 shell.exec，就必须限制到具体命令和参数。

- **批量任务里的审批疲劳**：一次给 20 个文件改名，Agent 逐个弹确认，用户点到第 8 个后开始无脑允许。更合理的是把同类操作合并成一次审批，或先允许 dry-run 输出计划，确认后连续执行。

- **测试/生产共用权限**：在测试环境允许 `docker rm -f`，生产环境忘了收紧。建议按环境拆分 profile，发布前检查。

- **只限制工具不限制参数**：允许 `grep` 没限制路径，可以读取到 `.env`、私钥或用户 token。工具描述里要写清楚“禁止访问哪些路径”，而不是只写能力。

- **工具描述太抽象**：如果 MCP 工具描述只写“执行命令”，模型无法判断风险。把副作用写进描述，例如 `Deletes files recursively. Requires approval when target is outside ./workspace.`，能显著减少误判。

## 可复用建议

- 默认 deny，最小权限起步。先把所有写操作设成 ask，跑一周看误报率，再逐步下沉到 allow。
- 按“副作用 + 可逆性”分级，不要只按工具名。`git push` 和 `git status` 风险完全不同，但都在 git 工具下。
- 给工具描述加结构化字段：`side_effect: low|medium|high|critical`、`requires_approval: true`、`reversible: true|false`。模型能利用这些字段判断，也能给审批流提供展示。
- 短期提权：平时只读，需要写时发一次性 token，系统在 30 分钟后自动回收。不要让长期 token 常驻在 Agent 配置里。
- 对外部渠道先发草稿或私聊，确认后再公发。消息一旦发出，就算能撤回也有时间差。
- 像管理 IAM 一样管理 Agent 权限。每次新增 MCP 工具或调整 allowlist，走配置评审，保留变更记录。

## 总结

权限边界不是一次配置完就结束的，它需要随工具数量、环境和任务风险持续迭代。一个可用的 Agent 不是“什么都能干”，而是“知道什么不能自己干，并且把需要人类确认的动作清晰、降噪地抛出来”。

做到默认安全、按需提权、全程可审计，Agent 的自动化程度才能真正提高，而不是靠运气。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/4330fa1bb5c60255.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/d5066061fbd7f783.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-25/6faab931ebe9967c.png)

