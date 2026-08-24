---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力
feedId: 34474
source: 综合讨论
publishedAt: 2026-08-24
---

## 背景

在 OpenClaw 或类似的 Agent 工程里，一个助手往往要同时兼顾代码生成、浏览器自动化、文件处理、消息推送等能力。早期做法很直接：把所有系统提示、工具定义、MCP 连接一次性注入上下文。

能力少的时候没什么问题，但一旦工具数量超过十几个，情况就开始变差：上下文窗口被大量占用，模型开始选错工具，响应变慢，维护也越来越痛苦。

Skills 机制想解决的就是这个问题：把“能力”拆成可描述、可发现、可加载/卸载的单元；助手先看到轻量索引，在需要时再加载完整指令和工具。

## 问题

全量注入主要有几个具体问题：

- **上下文膨胀**：每个 skill 的说明、示例、约束都堆在 system prompt 里，模型真正用于任务的空间被压缩。
- **工具选择退化**：当 MCP server 暴露 40+ 工具时，模型容易混淆相似工具，调用错误率上升。
- **指令冲突**：两个 skill 可能都定义了“如何读取文件”，但前提和边界不同。
- **维护困难**：改一个能力要重建全局 prompt，回归风险大。

按需加载不是银弹，但在多能力 Agent 上确实能把上下文从“全集”降到“工作集”。

## 做法/步骤

以我在 OpenClaw 中的实践为例，目录结构大致如下：

```text
skills/
  pdf-extract/
    manifest.yaml
    prompt.md
  web-automation/
    manifest.yaml
    prompt.md
```

`manifest.yaml` 里不写完整指令，只写触发线索和资源声明：

```yaml
name: pdf-extract
description: 从 PDF 中抽取文本、表格和元数据
triggers:
  - "解析 PDF"
  - "提取 PDF 表格"
  - "read pdf"
tools:
  - mcp:filesystem
  - builtin:shell
entry: prompt.md
ttl: 600
```

运行时流程如下：

1. 首轮先给模型一份“轻量索引”：只包含 skill 名称、一句描述、触发词。
2. 用户任务进来后，模型匹配到某个 skill，调用加载动作。
3. 系统把 `prompt.md` 注入上下文，同时按 `tools` 声明激活对应 MCP 工具。
4. 任务结束或空闲达到 `ttl` 后，卸载 prompt 和未使用的工具。

加载动作可以封装为 OpenClaw 的内置工具，比如 `load_skill(name)` 和 `unload_skill(name)`。模型自己决定何时加载，但配置里可以通过 `max_active` 限制同时加载数量。

与 MCP 的配合点在于：skill 不是替代 MCP，而是对 MCP 工具做了一层“分包”。MCP server 仍然负责连接，skill 决定哪些工具出现在当前上下文里。

## 踩坑点

- **触发描述写太宽**：比如 `description: "处理文件"` 会导致任何文件操作都命中，加载一堆不相关能力。建议用“具体动词 + 对象 + 场景”来描述。
- **工具声明不全**：skill 加载后调用工具时才发现没有权限或没有连接，首轮直接失败。最好在 manifest 里显式列出依赖，加载时做前置检查。
- **加载后不卸载**：上下文泄漏是最隐蔽的问题。短会话里无感，长会话会逐渐回到“全量注入”的状态。ttl 或任务结束钩子必须有。
- **循环依赖**：skill A 触发 skill B，B 又触发 A。可以加一个 `load_depth` 上限，或禁止嵌套加载。
- **版本漂移**：skill 的 prompt 改了，但触发词没同步，导致旧行为还在被命中。需要把 manifest 和 prompt 当作同一发布单元管理。

## 可复用建议

1. **先用窄触发**：宁可漏触发，也不要在初期追求高召回。漏了可以补触发词，误触发会污染后续判断。
2. **给每个 skill 写小样本集**：3-5 个应命中、3-5 个不应命中的任务描述，每次改触发词后跑一遍。
3. **记录触发日志**：哪个 skill 被加载、因为哪个词、是否成功调用工具、多久卸载。没有日志很难排错。
4. **设置资源上限**：最大并发 skill 数、单 skill 最大 token 数、最长存活时间。让加载变成可预测行为。
5. **高频能力直接固化**：如果一个 skill 在 80% 任务里都会用到，那它就不该按需加载，应该进默认 prompt。按需加载适合长尾能力。

## 总结

OpenClaw Skills 机制的核心不是“更聪明的 prompt”，而是把能力生命周期管起来：发现、加载、使用、卸载。对工程化 Agent 来说，上下文和工具面都是资源，按需加载是控制资源的手段。落地的关键是触发精准和卸载可靠，否则只是把全量注入换成了延迟注入。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/4713f08be112d410.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/22c4617b8df70658.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/c0ccc2822809628b.png)

