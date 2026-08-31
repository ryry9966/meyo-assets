---
title: AI 助手的 SOUL.md：给 Agent 设定人格和边界
feedId: 35532
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景

在 OpenClaw 里调试 Agent，经常遇到一种尴尬：跑 demo 很顺，但不敢把真实任务交给它。比如让它整理文件，它顺手改了目录权限；让它写脚本，输出风格从简洁变成话痨；遇到边界情况开始自由发挥，甚至调用不该调用的工具。

这些通常不是模型能力问题，而是缺少一个稳定的、随项目走的人格与边界文件。SOUL.md 就是用来解决这件事的：把 Agent 的身份、能力范围、行为边界、沟通风格和工具策略固化下来，可版本化、可复用、可测试。

## 问题

很多实践者把 system prompt 写在聊天框、配置文件或代码注释里，没有统一来源。结果是：

- 规则散落：不同会话、不同工具加载的规则不一致。
- 边界模糊：只说“注意安全”，但没定义哪些操作算危险。
- 无法版本化：改坏了难回滚，团队协作时更混乱。
- 风格漂移：Agent 在长任务中逐渐偏离初始设定。

## 做法/步骤

1. **在项目根目录创建 SOUL.md**，作为该工作区 Agent 的“宪法”。  
2. **分层编写**，建议包含以下模块：Identity、Capabilities、Boundaries、Communication、Tool Policy、Memory。  
3. **写具体规则，避免抽象描述**。示例：

```markdown
---
version: 1.2.0
scope: repo-ops
owner: platform-team
---

# SOUL.md

## Identity
你是本仓库的运维 Agent，使用中文，默认给出“结论 + 依据 + 风险”。

## Capabilities
- 可读取日志、检查服务状态、运行只读命令。
- 可修改配置，但必须先生成 diff。

## Boundaries
- 禁止执行：rm、dd、mkfs、shutdown、reboot、git push --force。
- 修改文件前必须展示 diff，超过 50 行必须请求人工确认。
- 不读取 .env、*.pem、id_rsa、token 文件。
- 不对外发送任何仓库内容。

## Communication
- 回复不超过 3 段，先结论后细节。
- 遇到不确定时明确说“需要确认”，不要编造。

## Tool Policy
- 优先使用只读 MCP 工具。
- 写操作必须引用本次任务上下文，不得凭记忆修改。
```

4. **在 OpenClaw 中加载 SOUL.md**：可以放在 Agent 初始 system prompt 之前，或通过文件引用注入。如果使用 MCP，也可以在工具描述中声明“遵循项目 SOUL.md”。关键是每次会话都自动加载，而不是靠人提醒。  
5. **验证边界**：写几条测试用例，比如诱导 Agent 执行危险命令、读取敏感文件、输出超长内容，看它是否拒绝或请求确认。

## 踩坑点

- **文件太长**：超过 200 行后，模型注意力稀释，后面的规则容易被忽略。核心规则放最前面，细节放附录。  
- **规则太抽象**：写“注意安全”等于没写。要具体到命令、路径、操作类型。  
- **人格设定过度**：让 Agent 扮演某个虚构角色，可能影响工具调用准确率。保持工程化，风格要求可以简单。  
- **敏感信息泄露**：不要把密钥、token、内网地址写进 SOUL.md。如果包含路径，用占位符。  
- **规则冲突**：例如“自主执行”和“写操作确认”冲突时，需要定义优先级，通常确认优先。  
- **未纳入版本管理**：SOUL.md 变更要走 git diff，避免“改了规则但队友不知道”。

## 可复用建议

- **按场景拆分**：SOUL.base.md、SOUL.ops.md、SOUL.dev.md，按需合并加载，避免单文件膨胀。  
- **用 YAML frontmatter 存元数据**：如 version、scope、owner，方便校验和追踪。  
- **把 SOUL.md 纳入 PR 评审**：规则变更可追溯，团队协作更安全。  
- **定期红队测试**：让另一个 Agent 尝试突破边界，根据结果补规则。  
- **记录边界触发日志**：哪些规则经常被触发，是否过于严格或宽松，再迭代。

## 总结

SOUL.md 的价值不是让 AI 更像人，而是让它在边界内稳定工作。把它当作配置文件来维护，而不是写一段漂亮的 prompt。对于 OpenClaw 用户来说，这是从“演示级 Agent”到“能托付任务的 Agent”的关键一步。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/cdde1d8cf298bc5b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/2c243d2941c0a0f7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/1d57d960fd8fb05a.png)

