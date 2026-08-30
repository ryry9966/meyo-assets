---
title: OpenClaw Skills 机制：从全量注入到按需加载的工程实践
feedId: 35480
source: 综合讨论
publishedAt: 2026-08-31
---

OpenClaw 上做 Agent 时，最容易出现的不是模型不够强，而是能力太多：系统提示里先塞一堆工具说明，再挂 MCP 服务，再加插件文档和示例。开发初期还好，等 PDF 解析、浏览器自动化、消息推送、数据清洗都进去后，上下文先被吃光。

Skills 机制要解决的就是这件事：把能力拆成独立目录，AI 不在一开始加载所有能力，而是根据任务元数据按需拉起相关 Skill。

## 问题：全量注入为什么不可持续

全量注入的代价不是线性可忽略的。每增加一个工具，不只是多一段描述，还会同时增加函数 schema、调用示例、错误处理规则。对 Agent 来说，真正有效的注意力被无关能力稀释，选择工具时更容易串。

MCP 也没有完全解决这个问题。MCP 让工具接入更规范，但如果所有 MCP 工具都同时暴露，问题和全量函数列表类似：schema 很长，模型不一定选得准。

Skills 更适合做“能力包”：包含说明、触发条件、脚本和参考文档。只有任务匹配时才加载，加载后也只把必要的操作说明放进上下文。

## 做法：一个 Skill 的最小结构

OpenClaw 里比较稳的目录结构是这样：

```text
skills/
  pdf-tools/
    SKILL.md
    scripts/
      extract_pdf.py
    references/
      pdf-schema.md
```

`SKILL.md` 不放完整文档，只放触发元数据和最短操作说明：

```markdown
---
name: pdf-tools
description: 提取 PDF 文本、表单字段与元数据；触发场景包括读取 PDF、解析发票、导出表格。
triggers:
  - 读取 PDF
  - 提取表单
  - 解析发票
---

# PDF 工具

## 可用脚本
- `extract_pdf.py --input file.pdf --mode text|form|meta`

## 输出约定
- 默认输出 JSON，单文件不超过 2000 token
- 大文件先输出摘要，不直接进上下文
```

按需加载的关键不是脚本能力，而是 `description` 与 `triggers` 的质量。它们就是检索信号。元数据写得好，模型才能在正确任务上拉起 Skill；写太宽，每次都被误触发。

加载后采取渐进式披露：`SKILL.md` 只保留热路径指令，复杂 schema、字段字典、错误码放到 `references/`。模型需要时再读，不需要就不占上下文。

## 踩坑点

第一是触发范围失控。`description: 处理文件` 这种写法几乎等于每次都会加载，和最早就塞进系统提示没有区别。一般建议写 3 到 5 个真实触发场景，同时写一个反例：例如“不要在处理图片时触发”。

第二是脚本输出污染上下文。脚本直接把 80 页 PDF 文本打印出来，比不加载还糟。脚本应当输出摘要或结构化的 JSON，超过 token 上限就写临时文件，只把文件路径和统计信息返回。

第三是路径与运行环境。脚本路径要相对 `SKILL.md` 解析，不要依赖全局安装的 Python 包。执行目录、虚拟环境、Windows 路径分隔符都要显式处理，否则换机器就挂。

第四是 MCP 和 Skills 重叠。某个能力已经由 MCP 服务提供时，不要再写一个同功能 Skill。通常做法是：MCP 负责动态外部数据，Skills 负责静态流程、领域知识和脚本编排。

## 可复用建议

- 一个 Skill 只解决一类任务，不要做成“工具箱”。
- 把 `description` 和 `triggers` 当成生产接口维护，而不是写完就不管。
- 脚本输出默认 JSON 或短文本，禁止无上限打印。
- 为每个 Skill 记录加载次数、加载 token 数、误触发次数。用一组固定任务测试，触发精度低于预期就先改元数据。
- 版本化 Skill 目录，升级时保留 `SKILL.md` 的兼容字段，避免老任务突然拉不到能力。

## 总结

Skills 不是新的模型能力，而是一种工程约定：把能力从“常驻上下文”变成“可检索、可加载、可约束”的包。它解决的是上下文开销和工具选择干扰问题。真正决定效果的是触发元数据、脚本输出约束和渐进式披露是否做到位。

在 OpenClaw 里用 Skills，比较合理的顺序是：先把高频能力拆出去，跑通一个 Skill 的触发和脚本调用；再逐步迁移低频能力；最后用加载日志和任务集做回归。这样比一次性重构所有提示词风险低得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/5c1d351369d7cc41.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/4c98413f2484f9e1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/ad170c49b14d777b.png)

