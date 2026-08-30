---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 35483
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景

在 OpenClaw 的 Agent 实践中，能力膨胀是一个很常见的问题。接入十来个插件、脚本或自动化流程后，每次对话都要携带大量提示词、工具描述和操作步骤，结果是上下文变长、首字延迟上升、模型误触发增多。

Skills 机制要解决的核心不是“装更多能力”，而是 **默认不加载，按需再注入**。它把技能拆成两层：

- 索引层：只保留技能名称、用途、触发条件、权限要求，供路由判断；
- 执行层：命中后再加载完整指令、脚本路径和参考文档。

这样，未命中的技能几乎不占上下文，也不会干扰当前任务。

## 典型问题

常见症状包括：

- 装了 20 个技能后，助手开始“串味”，日志分析里混入客服话术；
- 明明只问天气，却加载了文件整理、命令执行等多个无关技能；
- 每次会话 token 消耗高，但大部分能力从没被真正用过；
- 多个技能描述相近，路由时同时命中，互相污染上下文。

这些问题通常不在于模型能力差，而在于技能没有做轻量索引和触发边界控制。

## 做法与步骤

OpenClaw 中，技能一般以目录形式存在，核心文件通常类似 `SKILL.md` 或技能清单。一个可复用的结构如下：

```text
skills/
  log-analyzer/
    SKILL.md
    scripts/
      parse_logs.py
    references/
      patterns.md
```

`SKILL.md` 的元信息需要区分“给路由看的描述”和“给模型加载后的正文”。例如：

```yaml
---
name: log-analyzer
description: >
  Use when the user asks to analyze recent error logs, find stack traces,
  or summarize crash patterns. Not for general file reading.
tools:
  - scripts/parse_logs.py
permissions:
  - read:logs
---
```

运行时的加载流程大致是：

1. OpenClaw 启动时扫描 `skills/`，只读取元信息，构建轻量索引；
2. 用户输入进入路由层，根据 `description`、触发条件和当前上下文匹配技能；
3. 命中后加载该技能的完整正文、可用脚本列表和权限约束；
4. 未命中则不注入任何技能正文。

这里的关键点是 **description 必须足够窄**。它不是给用户看的简介，而是给路由器的匹配规则。

## 踩坑点

1. **触发条件写得太宽泛**  
   像“帮助用户处理文件”这种描述，会导致大量无关任务都命中该技能。解决方式是同时写 `When to use` 和 `When not to use`，明确负向边界。

2. **多个技能同时命中**  
   技能名或描述相似时，路由可能会同时加载两个甚至三个技能。可以给技能增加优先级、互斥标签或更严格的触发短语，避免上下文互相污染。

3. **修改技能后旧索引不失效**  
   改完 `SKILL.md`，运行中的会话仍使用旧索引。OpenClaw 需要触发 reload skills，或者新开会话验证。调试时建议固定一个测试 prompt，观察路由是否重新加载。

4. **脚本路径和参数处理不当**  
   如果技能正文直接拼接文件路径或外部输入去执行命令，容易出现路径穿越或命令注入。脚本参数应做白名单校验，技能只暴露有限命令，不要把任意 shell 权限交给模型。

5. **正文太长导致加载后 token 膨胀**  
   即使按需加载，一次注入几 KB 的正文仍然很重。参考文档应外置到 `references/`，让模型在需要时再读取，而不是全量塞入 `SKILL.md`。

## 可复用建议

- **单一职责**：一个技能只解决一类任务，正文尽量控制在一屏以内；
- **描述结构化**：固定写 `When to use / When not to use / Required permissions`；
- **用测试集验证路由**：准备 10 组 prompt，分别验证该触发和不触发的情况，记录加载的技能与 token 变化；
- **纳入版本管理**：技能目录进 git，变更后显式 reload；
- **与 MCP 分工**：外部 API、数据库、浏览器等强交互用 MCP；确定性本地流程、规范约束、脚本编排更适合用 Skills。

## 总结

OpenClaw Skills 的价值不在于让助手“知道更多”，而在于让能力可索引、可路由、可测试。先管住描述层和触发边界，再考虑脚本编排，往往比堆更多技能更有效。

如果你在 OpenClaw-CN 社区里做自动化实践，不妨先检查现有技能的 description 是否过宽，以及是否有多个技能经常被同时触发。把这些问题收干净，很多诡异行为会自然消失。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/f445ac8e5e3d6206.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/e90eae5a6fb7b0fe.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/86723e833e7f88bc.png)

