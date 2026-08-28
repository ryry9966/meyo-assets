---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 35100
source: 综合讨论
publishedAt: 2026-08-28
---

## 背景：Agent 能力膨胀的代价

在 OpenClaw 里接入 MCP、插件和本地脚本后，最容易出现的情况是：系统提示词越写越长，所有工具一股脑挂到会话里，结果模型开始“乱选工具”，响应变慢，上下文也被无关能力占满。

Skills 机制解决的不是“能不能做”，而是“什么时候加载什么”。它把一组能力封装成独立模块，路由层只保留轻量描述，真正执行任务时再按需注入指令和工具。这种设计更适合长期运行的 Agent，尤其是同时挂载多个 MCP server 或插件时。

## 问题：全量加载的典型症状

一个典型场景：Agent 接了文件处理、浏览器自动化、数据库查询、消息推送。用户只是问“把这份 PDF 总结一下”，但系统里同时挂着浏览器工具和 SQL 工具。结果模型可能误判，先去调用浏览器；或者因为工具定义太多，消耗大量上下文空间，降低指令遵循能力。

另一个问题是上下文预算。很多 MCP 工具声明本身就不小，再加上各类插件的使用说明，几万 token 很轻松被吃掉。全量加载等于为用不到的能力持续付费。

## 做法：把能力拆成可路由的 Skill

OpenClaw Skills 的基本结构可以是：

```text
skills/
  pdf-summarize/
    SKILL.md
    skill.yaml
    scripts/extract_pdf.py
    references/citation_rules.md
```

其中 `skill.yaml` 只放轻量元数据：

```yaml
name: pdf-summarize
description: 当用户要求总结 PDF、合同、论文或提取 PDF 核心内容时触发。
triggers:
  - "总结 PDF"
  - "提取 PDF"
  - "pdf 摘要"
tools:
  - read_pdf
  - chunk_text
permissions:
  - read:files
```

启动时只注册 `name`、`description`、`triggers` 到路由层，不加载 `SKILL.md` 正文，也不挂工具。运行时根据用户输入匹配 triggers 或做 embedding 相似度计算，命中后再把 `SKILL.md` 注入系统上下文，挂载对应工具。任务结束或用户切换主题后卸载，回收 token。

对于长参考文件，不要让模型一次性读取全部内容。`SKILL.md` 只写工作流，具体规则、示例、API 文档放在 `references/` 下，由模型在需要时用 `read_file` 自行取用。这样既保留能力，又避免无谓消耗。

## 踩坑点

第一是描述质量差导致不触发或误触。`description` 如果写成“处理文件”，几乎所有文件问题都会命中，反而干扰路由。应该写清边界，例如“适用于 PDF、合同、论文的文本提取与摘要，不适用于表格数据分析”。

第二是多个 Skill 同时命中。比如“总结 PDF”和“长文摘要”都可能被激活。建议设置优先级和互斥规则，一次只激活一个主 Skill，最多叠加一个辅助 Skill，避免上下文争抢。

第三是工具命名冲突。本地脚本和 MCP 工具可能都叫 `read_file` 或 `search`。加载后互相覆盖会导致调用错误。最好使用命名空间，比如 `pdf.read_file`、`web.search`，从源头避免冲突。

第四是卸载不彻底。如果上一轮激活了浏览器 Skill，下一轮用户问数据库问题，但旧指令还留在上下文里，模型仍可能被带偏。建议在每轮任务开始前做上下文裁剪，明确移除已结束 Skill 的指令和工具。

第五是权限过度。Skill 声明了 `write:files`，但实际只读 PDF，一旦被误用风险很大。按 Skill 做最小权限声明，敏感操作执行前二次确认。

## 可复用建议

- 把 `description` 当检索文档写：前两句说明“用户什么意图下应该触发”，后面再写“它能做什么”。
- 坚持两级加载：一级 `SKILL.md` 放工作流，二级 `references/` 放细节，按需读取。
- 建立固定测试集，用一批真实话术跑回归，看路由是否命中、工具是否正确挂载、上下文是否回落。
- 记录每次 Skill 激活原因和 token 变化。如果激活后上下文涨太多，说明细节没有做到按需加载。
- 给 Skill 加版本号，灰度发布。改完 `SKILL.md` 后注意路由缓存是否失效，可以用 content hash 作为缓存键。

## 总结

OpenClaw Skills 不是简单地把提示词拆成文件，而是把“能力路由”和“上下文预算管理”结合起来。轻量声明负责判断该不该加载，正文和工具只在需要时进入会话，用完即走。这个思路对 MCP、插件、本地脚本同样适用。关键不在目录结构，而在描述是否精准、工具是否隔离、卸载是否彻底、加载是否可观测。把这些做扎实，Agent 才不会随着能力变多而变笨。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/9a7716b15ad27509.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/2686c7c8da81ef7d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-28/56074941961479e7.png)

