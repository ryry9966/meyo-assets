---
title: 给 Agent 装上“灵魂”与“护栏”：SOUL.md 设计与落地方案
feedId: 31154
source: 综合讨论
publishedAt: 2026-08-01
---

## 背景：失控的 Agent 与缺失的“人格文件”

在基于 OpenClaw 搭建自动化 Agent、MCP 工具链和插件体系的过程中，多数人习惯用一段 System Prompt 来定义助手行为。随着能力边界扩大——Agent 开始连接数据库、操作文件、触发流水线——一段自然语言描述越来越不可控。不同开发者改动的 Prompt 相互覆盖，边界条件散落各处，最终 Agent 要么过于保守，要么越权操作，甚至把内部信息吐给外部工具。

我们需要一种工程化的方式，将 **人格（Personality）与边界（Boundaries）** 显式定义在单一事实来源中，版本化管理，并通过插件注入到每次会话。我把这个文件称为 **SOUL.md**——不是“灵魂”的文艺说法，而是 **System Operating & Usage Limits** 的简写（也恰好符合精神内核）。

## 问题：模糊的 System Prompt 无法约束复杂 Agent

常见困境：

- **人格不一致**：同一个 Agent 在对话中温和，在 Slack 通知里冷漠，在日志里又像工程师。
- **边界隐式**：“不要泄露 API Key”写在某处，“不能删除生产数据”写在另一处，回溯困难。
- **权限与语气混在一起**：把工具使用限制、响应风格、合规要求全挤在一段文本里，LLM 很容易忽略或误解。
- **团队协作噩梦**：产品经理想调语气，SRE 要加安全规则，双方的修改频繁冲突。

SOUL.md 的初衷就是对这些问题建模，让非技术人员也能维护人格部分，而工程侧专心维护边界和工具约束。

## 做法：设计并集成 SOUL.md

### 1. 文件结构
采用 Markdown + YAML frontmatter，便于解析和分层加载。

```markdown
---
identity:
  name: "OpsBot"
  role: "Site Reliability Engineering Assistant"
tone:
  style: "concise, professional"
  humor: false
  emoji: minimal
boundaries:
  data_leakage: "Never output credentials, tokens, or internal IPs"
  tool_restrictions:
    - "rw only on /tmp workspace"
    - "no destructive commands without confirmation"
  domain_knowledge: "Only answer from provided runbooks and documentation"
memory:
  type: "episodic_summary"
  retention: "last 5 interactions per task"
tool_permissions:
  mcp_servers:
    - name: "filesystem"
      allow: ["read"]
    - name: "slack"
      allow: ["post_message"]
---
```

正文部分放 Few-Shot 示例或更细腻的角色说明，例如如何处理故障升级、如何拒绝不合理请求的话术。

### 2. 注入方式（OpenClaw + MCP）
利用 OpenClaw 的会话钩子把 SOUL.md 注入 System Prompt。具体做法：

- 在 Agent 初始化时，通过一个轻量 MCP 资源暴露 SOUL.md，读取 frontmatter 并构建结构化 System Prompt。
- 将 `boundaries` 转化为约束性指令，插入工具调用前的策略检查；必要时用代码层做二次拦截（如解析工具调用参数是否合规）。
- 对 `tone` 和 `identity` 部分，生成简洁声明，避免占用过多上下文。

代码层拦截比纯模型遵循更可靠，比如文件系统工具调用前，检查路径是否在 `/tmp` 下，否则直接返回拒绝，不依赖模型自觉。

### 3. 分层覆盖
一个基础 SOUL.md 定义企业通用规则，特定 Agent 可叠加增量文件：

```yaml
extends: "base/soul-ops.md"
tone:
  humor: true  # 覆盖为允许幽默
```

工具加载时合并配置，确保基础边界不被削弱。

## 踩坑点

### 1. 人格描述越详细，遵循度反而下降
很多人把 SOUL.md 写成一篇小说角色设定，但实际测试表明，超过 800 字符的人格描述会让 LLM 顾此失彼。保持 **identity** 在 3~5 句内，**tone** 用关键词而非段落描述。用 Few-Shot 示例代替冗长的“你应该怎样说话”。

### 2. 边界指令与工具描述冲突
如果 MCP 工具描述包含“删除文件”，而 SOUL.md 说“禁止删除文件”，模型常被工具描述拉走。我们的解法是：**工具描述本身不能包含超出边界的能力**——在工具注册时就用过滤器裁剪，只暴露被允许的操作。MCP Server 侧做能力收缩，比在 Prompt 里打补丁有效。

### 3. 上下文快速膨胀
把整个 SOUL.md 全文塞进 System Prompt 会吃掉宝贵的 Token。采用 **延迟加载**：只在需要时引用对应段落；或把静态边界编译成规则引擎，Prompt 中只放一句“你的边界已在系统层硬限制，请不要尝试绕过”。

### 4. 版本漂移与测试
修改 SOUL.md 后很难手动验证所有场景。我们为边界部分编写了语义化测试：给定一批越权请求，验证 Agent 是否拒绝。集成到 CI 中，每次 PR 自动跑。

## 可复用建议

- **最小人格，硬边界**：人格用示例承载，边界用代码承载，忌让模型自行裁决安全。
- **结构化优先**：YAML frontmatter 方便程序解析，不要把所有内容揉成一团自然语言。
- **分层继承**：用 `extends` 机制区分基础规则和 Agent 个性，避免复制粘贴。
- **结合 MCP 策略**：在 MCP Gateway 层实现工具筛选，让 Agent 压根看不到不可用的工具。
- **版本管理，审计日志**：把 SOUL.md 提交到 Git，每次 Agent 执行关键操作时记录所用版本，方便回溯。
- **可测试性**：用断言验证边界是否生效，不要让“护栏”只停留在纸面上。

## 总结

SOUL.md 并不是什么魔法，而是一种**把 Agent 的人格和边界工程化**的实践。它赋予团队一个单一事实来源，使行为可预测、可审计、可继承。在 OpenClaw 这类高度可扩展的 Agent 框架里，配合 MCP 的工具过滤和代码层拦截，才能真正把“安全”和“个性”落地为系统的一部分，而不是寄希望于一段 Prompt。

当你下次发现 Agent 用自己的幽默调侃了生产事故时，别急着调 Prompt，先看看它的 SOUL.md 里有没有写 `humor: false` —— 以及，是否真的执行了。

---

