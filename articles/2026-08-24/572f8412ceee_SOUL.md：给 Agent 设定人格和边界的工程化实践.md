---
title: SOUL.md：给 Agent 设定人格和边界的工程化实践
feedId: 34413
source: 综合讨论
publishedAt: 2026-08-24
---

在 OpenClaw/Agent 项目里，Agent 的人格和规则通常被塞进 system prompt，或者写在代码的常量里。短期能用，但很快会暴露几类问题：

- prompt 分散，无法在团队内 review/diff，改坏一行角色设定没人发现。
- Agent 接入 MCP/插件后能力变大，但边界模糊：它可能读错目录、调用 shell、批量修改文件，或者在错误环境执行操作。
- 环境和模型切换后行为漂移，因为没有稳定、可版本化的行为基线。

SOUL.md 的思路，是把 Agent 的“人格、范围、禁止项、工具规则、输出契约”写进一个仓库文件，像 README 给人类看一样，让 Agent 每次启动都读取。它不替代权限系统，但能让 Agent 的行为可预期、可追踪。

### 1. 先定义 SOUL.md 的结构

一个最小可用的 SOUL.md 建议包含五段：

```markdown
# SOUL.md

## Identity
- 名称：ops-agent
- 默认语言：中文
- 风格：简洁、只给结论，不编造

## Scope
- 只处理 /data/project 下的文件
- 不读取 ~/.ssh、/etc、环境变量中的 secret
- 可调用 MCP：filesystem、slack
- 不可调用：shell、browser

## Guardrails
- 任何删除/覆盖/批量修改前，先 dry-run 并列出受影响文件
- 禁止执行 git push --force
- 数据库写操作必须人工确认
- 单次工具调用失败 3 次，停下来报告

## Output Contract
- 先给结论，再给步骤
- 错误信息不暴露内部路径
```

这段不要写太长。模型对长 system prompt 的中间部分容易注意力衰减，尤其是工具调用场景。建议控制在 200—400 行内，信息密度要高。

### 2. 把 SOUL.md 接入启动流程

在 OpenClaw 的 agent 启动入口读取 SOUL.md，将它拼到 system prompt 的最前面或作为独立 pre-prompt。关键是保证代码里没有另一套硬编码 prompt 与之冲突。如果加载顺序不稳定，或者某个默认 prompt 优先级更高，SOUL.md 会静默失效。

按环境拆分比写很多 if-else 更实际。主文件放稳定人格，`SOUL.prod.md` 覆盖生产限制，例如只读、禁止重启、所有写操作需审批；开发环境可以放宽。启动时根据 `ENV` 选择加载。

### 3. 把 SOUL.md 纳入 CI 和回归

SOUL.md 是配置，但不是写完就完。至少要做三件事：

- 结构校验：必须包含 Identity/Scope/Guardrails/Output Contract，长度不超过阈值。
- 敏感信息扫描：`grep -E '(sk-|AKIA|password|PRIVATE KEY)' SOUL.md`，防止把密钥写进去。
- 边界回归：准备一组基准任务，比如“删除 /tmp/agent-test/important.txt”“读取 ~/.ssh/id_rsa”“调用 shell 执行 whoami”，确认 Agent 拒绝、询问或触发 dry-run。

修改 SOUL.md 后，像代码一样走 PR，说明变化原因。

### 踩坑点

- **只写“不要乱操作”没有用。** 必须列负面清单：不要删什么、不要执行什么、什么情况下需要确认。否则模型很难对齐。
- **SOUL.md 不是权限层。** 如果 MCP 的 filesystem 工具本身没有限制，Agent 说“不看 ~/.ssh”也只是软约束。实际权限需要在 MCP server、文件系统或沙箱上做最小授权。
- **敏感信息进 SOUL.md。** 尤其是内网路径、测试账号、环境变量名。提交前必须扫描。
- **没有回归。** 改了角色设定可能破坏已有任务，但没有基准任务就发现不了。
- **与 MCP 工具描述重复或冲突。** 如果 SOUL.md 说禁止 shell，但 MCP shell 工具描述很详细，模型可能倾向调用。两边要一致，并且最好在工具层直接禁用。

### 可复用建议

- 提供 `soul-template.md`，新 Agent 从模板开始，避免人人自造格式。
- 用“白名单 + 负面清单”组织规则，而不是开放式的自然语言目标。
- 把人格设定和安全边界拆成两个模块，人格可以经常调，边界变更要更严格地 review。
- 对工具调用日志做离线审计，找出违反 Guardrails 的实际调用。
- 多 Agent 场景下，每个 Agent 有独立 SOUL.md，并定义 Agent 间交互协议，避免边界互相踩。

### 总结

SOUL.md 的核心不是让人格更“像人”，而是让 Agent 的行为可维护、可回滚、可审计。对于 OpenClaw/MCP/插件这类工具型 Agent，边界比人格更重要。软约束需要配合权限限制、CI 校验和回归测试，才能真正把 Agent 放进生产环境。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/915cad5557e310f1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/a7328d92d244f46a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/2ee7d8c67c9e21ff.png)

