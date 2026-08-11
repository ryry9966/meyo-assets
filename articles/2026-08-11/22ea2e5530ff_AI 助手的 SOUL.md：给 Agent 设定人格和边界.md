---
title: AI 助手的 SOUL.md：给 Agent 设定人格和边界
feedId: 32510
source: 综合讨论
publishedAt: 2026-08-11
---

# AI 助手的 SOUL.md：给 Agent 设定人格和边界

## 0. 这篇在说什么

给 AI 助手写人设并不是新鲜事，但在工程化 Agent 系统里，如何**结构化地定义人格、能力和行为边界**，并让它稳定地作用于工具调用、记忆和对话流程，仍然是一块缺少标准方案的地方。

在 OpenClaw 生态中，Agent 往往要对接 MCP 工具、执行自动化动作、读写长期记忆。这时“人设”就不再只影响语气，它会直接影响**安全性、工具调用权限、上下文的决策倾向**。于是我尝试把这块抽象成一个独立文件：`SOUL.md`，并让它和 Agent 的生命周期绑定。

这篇文章不讨论“如何写出更有趣的角色”，而是聚焦工程侧：**怎么定义、怎么加载、怎么保证边界不崩塌**。

---

## 1. 背景：为什么需要 SOUL.md

最早我只是在 System Prompt 里用一段自然语言写行为指导，但它有三个明显问题：

1. **长上下文稀释**：当工具返回大量数据或对话变长，靠前的人设描述会被注意力机制削弱，导致“性格漂移”。
2. **边界模糊**：自然语言描述难以约束 Agent 的权限边界。例如“只允许操作 `sandbox/` 目录”，却可能因为上下文误导而走出这个范围。
3. **不可测试**：无法针对“人设是否生效”做自动化验证。

于是考虑把人格定义和系统指令分离，成为一个结构化配置文件，既参与渲染，又可以被程序校验。

`SOUL.md` 的设计目标：

- 声明式定义 Agent 的**身份、能力、知识域、语气、行为边界**
- 提供结构化字段，能在 Prompt 模板里精确注入
- 支持版本化和自动化测试

---

## 2. 结构设计：一个可被解析的 SOUL.md

核心思路：用 Markdown + 可识别的章节标识，让人类可读的同时能被解析器提取字段。示例如下：

```markdown
# SOUL

## identity
name: ops-buddy
role: 运维助手
persona: 冷静、直接的工程师，不爱废话
language: zh-CN

## capabilities
- 查询服务器状态
- 执行受限 shell 命令
- 读写 sandbox 文件
- 创建 / 管理定时任务

## boundaries
- 拒绝执行破坏性命令
- 文件操作限制在 /sandbox
- 不泄露系统级环境变量
- 不猜测未授权的信息

## style
tone: 工程化、简洁
format: 输出使用 Markdown，错误使用 fenced block

## memory
- 会记住用户的部署偏好
- 不主动记录密码或 token
```

解析器可以在启动时加载这个文件，把 `identity`、`boundaries` 等字段映射到不同类型的 Prompt 段——例如 Identity 固定在最前，Boundaries 会在涉及工具调用时生成额外的 Safety 片段。

这样设计的好处是：**人设是配置，不是代码**，方便非开发人员调整，也方便在 CI 里跑 lint。

---

## 3. 做法/步骤：让 SOUL.md 作用到 Agent 上

以 OpenClaw 的一个典型 Agent 实现为例，完整流程大致如下：

### 3.1 加载与解析

```ts
import { readFileSync } from 'fs';
import matter from 'gray-matter'; // 也可以用简单正则解析

const soulRaw = readFileSync('./agents/ops-buddy/soul.md', 'utf-8');
const { data, content } = matter(soulRaw);
// data 里是 frontmatter，content 是正文，也可以完全基于章节解析
```

我通常不用 frontmatter，而是用章节解析器（如 `marked` + 自定义 renderer），解析出每个 section 的纯文本，作为可注入的片段。

### 3.2 注入到 System Prompt

根据任务类型动态拼接：

- **Identity section** → System Prompt 头部
- **Boundaries section** → 插入到工具调用说明之后，作为硬约束
- **Style section** → 放在输出格式指导区

示例模板片段：

```
# 身份
{{soul.identity}}

# 可用工具
{{tools_description}}

# 行为边界（硬约束，不可违反）
{{soul.boundaries}}

# 输出格式
{{soul.style}}
```

### 3.3 与工具调用联动的安全层

仅靠 Prompt 里的文字约束是不够的。`boundaries` 需要转化为运行时检查：

- 在 **工具调用前（pre-invoke hook）**，检查调用参数是否超出边界
- 在 **工具调用后（post-invoke hook）**，检查返回内容是否包含敏感信息
- 违反边界时，Agent 返回一个标准化的拒绝消息，而不是直接将工具错误暴露给用户。

示例 tool guard：

```ts
function sandboxGuard(toolName: string, args: Record<string, any>): boolean {
  if (toolName === 'write_file' || toolName === 'read_file') {
    const target = path.resolve(args.path);
    return target.startsWith('/sandbox');
  }
  return true; // 其他工具另行判断
}
```

### 3.4 记忆与人格的持久化

Agent 和用户的互动风格需要在长期记忆中保持一致。我让 Memory 模块在写入时也参考 `SOUL.md` 的 `style` 和 `memory` 字段，避免记忆条目本身带偏风格。例如，记忆总结会要求“保持简洁工程化语气”。

---

## 4. 踩坑点

### 4.1 边界约束在长对话中失效

即使把 boundaries 放在 System Prompt 后半段，当工具输出很大时，模型仍可能忽视它们。解决方案除了运行时 guard，还要在每次做出可能越界的决策前，触发一个内部的“核查回合”，用精简 Prompt 二次确认。

### 4.2 人格描述过于冗长

早期 SOUL.md 写成了“角色小传”，结果挤占了大量 token。后面硬性规定 `identity` 不超过 150 字，`boundaries` 每条不超过一行。约束精确度的最好办法是限制长度。

### 4.3 不同模型对自然语言边界的遵守程度不同

开源小模型对硬边界指令的执行力远弱于 GPT-4 类。实践中只能做 **防御性设计**——假设模型随时可能尝试越界，因此 runtime guard 是必须的，不能只靠 Prompt。

---

## 5. 可复用建议

1. **边界定义要量化**：不要写“尽量少用资源”，而是“CPU 使用超 80% 时禁止启动新进程”。
2. **SOUL.md 纳入版本控制**：人设调整经常是调试行为的一部分，保留修改历史可以避免“改了人设就自愈”的假象。
3. **写自动化测试**：对 SOUL 解析结果做 snapshot 测试；用模拟对话验证 Agent 在边界场景下是否输出拒绝信息。
4. **与 MCP server 的 manifest 联动**：工具能力和边界是两面。SOUL 里的 `capabilities` 应该能从 MCP 的 tool list 自动生成基础骨架，再人工添加约束。
5. **为不同环境准备 profile**：开发环境和生产环境的 Agent 边界可以不同（例如生产不允许任意写文件），可以用 `soul.production.md` 这种方式覆盖。

---

## 6. 总结

`SOUL.md` 本质上是一套 **Agent 人格与边界的结构化声明**，它解决的不是“让 AI 更可爱”的问题，而是：

- 人设不被长上下文冲淡
- 行为边界可被程序化校验
- 人格定义可测试、可版本化

在 OpenClaw 的自动化实践中，人格不仅是 flavor，更是安全面。把人格写进代码旁边，用配置管理它，再用 guard 兜底——这样出来的 Agent 才可能真正稳定地听从你的“灵魂”。

希望这个思路能给你在构建自己的 Agent 时提供一点参考。

---

