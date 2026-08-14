---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 33071
source: 综合讨论
publishedAt: 2026-08-14
---

# OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份

很多 OpenClaw 用户一开始会把 agent 当成一个“大 prompt 容器”：默认语气、工具偏好、禁止事项、工作流约定全部塞进 system prompt。刚开始能跑，但不出两周就会出现三个典型问题：上下文窗口被身份描述吃掉一大块；改一个行为偏好要翻好几处配置；多 agent 或多工作区之间很难复用同一套行为标准。

IDENTITY.md 是我最近在 OpenClaw 工作区里稳定下来的一种实践。它把“AI 是谁、怎么干活、哪些事不做、最近学到了什么”从临时 prompt 中剥离出来，变成一个可版本化、可 diff、可进化的工程文件。

## 背景

OpenClaw 不是一次性问答工具，而是常驻后台、通过 MCP/插件调用外部系统、执行自动化任务的 agent。这类 agent 的行为一致性很重要。比如你训练它“工具失败时最多重试两次”，如果没有一个稳定的身份层，某天你改了 system prompt，这个规则就可能丢失。或者你在两个工作区里想让 agent 保持同样的基础语气，只能复制粘贴 prompt，很快两边就不一致了。

IDENTITY.md 的目标很简单：给 agent 一个单一事实来源（source of truth），让它知道自己默认该怎么行动，同时允许这个“身份”随时间进化。

## 做法/步骤

### 1. 建立 IDENTITY.md 文件

在 OpenClaw 工作区根目录下新建 `IDENTITY.md`，不要放到某个插件目录里。建议加上 YAML frontmatter 记录版本和适用范围：

```markdown
---
version: 1.3.0
last_reviewed: 2025-04-01
scope: global
---

# Core Identity
- 你是运行在 OpenClaw 中的自动化助理。
- 默认语气：简洁、工程化，不输出恭维。
- 禁止：编造工具输出；修改工作区以外的文件路径。

# Operating Principles
- 执行多步骤任务前，先列出简短 plan。
- 工具调用失败时，最多重试 2 次，然后返回结构化错误。
- 涉及删除操作时，必须先请求用户确认。

# Tool Usage Policy
- MCP weather 工具结果必须原样引用，不得补充未返回的数据。
- 文件系统工具只能操作 ./workspace 下的路径。

# Evolution Log
## 2025-04-01
- 增加 weather 工具缓存策略：同一城市 10 分钟内不重复调用。
```

### 2. 在 OpenClaw 启动时加载

OpenClaw 支持在全局配置中指定 identity 文件。我用的是 YAML 配置：

```yaml
identity_file: ~/.openclaw/workspace/IDENTITY.md
load_strategy: on_session_start
reload_command: /identity-reload
```

这样每次新会话启动时，OpenClaw 会把 IDENTITY.md 作为行为基线加载，而不是全部塞进 system prompt 的头部。如果你的 OpenClaw 版本不支持 `identity_file` 字段，也可以在 system prompt 里写一句：

```text
You are an OpenClaw agent. Load and follow the identity defined in IDENTITY.md.
If there is a conflict between this prompt and IDENTITY.md, safety rules in this prompt win.
```

但我不建议长期靠 system prompt 引用，因为这样身份文件的内容还是会被完整注入上下文，只是换了位置。更好的做法是让 OpenClaw 的 identity loader 只加载相关段落，或者用片段引用。

### 3. 让身份“可进化”

Evolution Log 是这套实践里最关键的部分。不要只是把身份文件当成静态配置，每次你发现 agent 反复犯同一个错误，或者你调整了它的工作方式，都应该往 Evolution Log 里追加一条记录。比如：

```markdown
## 2025-04-06
- 发现 agent 在 MCP 插件超时后会继续尝试其他工具，导致任务卡死。
- 新规则：任何工具调用超时后，立即停止当前任务并返回 timeout 状态。
```

同时 bump 版本号。这样你可以在 git 历史里看到 agent 的“成长轨迹”。

## 踩坑点

**1. IDENTITY.md 变成杂货铺**
一开始容易把所有 prompt 都搬进去，包括具体任务的指令、临时调试信息。这样 identity 文件会膨胀到几千行，每次加载反而增加上下文成本。我的判断标准是：如果一条规则只适用于某一个具体任务，就留在任务 prompt 里；如果它描述的是 agent 的默认行为，才放进 IDENTITY.md。

**2. 版本不更新导致缓存旧身份**
OpenClaw 有些版本会缓存 identity 文件。改完 IDENTITY.md 后如果忘记 bump version，旧会话可能还在用缓存内容。一定要在 frontmatter 里更新 `version`，并且配置 `reload_command` 手动触发重载。

**3. 与 system prompt 冲突**
全局 system prompt 和 IDENTITY.md 之间难免有重叠。比如 system prompt 里写了“语气友好”，IDENTITY.md 里写了“语气简洁”。如果不定义优先级，agent 会随机选择。建议在 system prompt 里明确：安全规则优先，风格规则以 IDENTITY.md 为准。

**4. 敏感信息入库**
不要把 API key、私有路径、内部域名写进 IDENTITY.md，尤其是如果你把这个文件提交到 git。身份文件应该是行为定义，不是凭证管理。

**5. 用抽象词代替可验证行为**
“要聪明”“尽量别出错”这类描述没有任何约束力。要写成可验证的行为，比如“失败时返回 JSON，包含 error_code 和 retry_allowed 字段”。

## 可复用建议

- **静态 + 动态分层**：IDENTITY.md 只放稳定规则；每次任务中产生的经验先写到 `memory/evolution.md`，每两周 review 一次，把真正稳定的经验提升进 IDENTITY.md。
- **用 git 管理**：每次修改 IDENTITY.md 都要提交，commit message 写清楚为什么改，而不是“update identity”。
- **行为测试**：写几组固定问题验证身份是否生效。比如问“今天北京天气怎么样”，agent 应该调用 MCP weather 工具而不是编造。修改 IDENTITY.md 后跑一遍这些测试。
- **多 agent 复用**：如果同时跑多个 OpenClaw agent，可以把 Core Identity 拆成共享模板，每个 agent 只维护自己的 Evolution Log。这样既保持一致性，又保留个体差异。
- **定期清理**：每两周检查一次 Evolution Log，把过时规则删除或合并。不要让旧规则一直堆着。

## 总结

IDENTITY.md 不是另一个 prompt 文件，而是把 agent 的“身份”从临时提示升级为可维护、可版本化的工程资产。它解决的不是“怎么写 prompt”的问题，而是“当 agent 跑了一个月之后，你如何保证它仍然按你的预期工作”。对于 OpenClaw 的重度用户来说，这套实践比频繁微调 system prompt 更可持续。

---

