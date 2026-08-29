---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 35179
source: 综合讨论
publishedAt: 2026-08-29
---

在 OpenClaw 里，Agent 很容易变成一个“每次都像新人”：任务重启后偏好丢失、工具调用风格漂移、协作时互相覆盖边界。把 personality 写在 system prompt 里能解决一部分，但很快就会遇到两个问题：prompt 太长挤压工具定义；身份更新没有版本，越改越乱。

IDENTITY.md 的思路是把身份从 prompt 中抽出来，作为一个独立、可维护、可进化的文件。它不只是角色卡，而是 Agent 运行时的最小身份锚点。

## 问题

一个典型的 OpenClaw agent 会同时做任务规划、调用 MCP 工具、维护本地文件、偶尔需要人确认。如果没有稳定的身份层，常见表现是：

- 每次会话都要重新学习用户偏好；
- 同一仓库的不同 agent 实例对“能否修改文件”有不同理解；
- 长期任务跑久了，行为逐渐偏离初始目标；
- 插件或工具误用后，无法判断是边界不清还是 prompt 冲突。

这些问题单靠增加 system prompt 长度解决不了，反而会挤压可用上下文。身份信息需要分层：核心稳定、操作可调、记忆外置。

## 做法

我目前的做法如下。

### 1. 把身份文件独立出来

在 agent 工作目录放一个 `IDENTITY.md`，在 OpenClaw 的 bootstrap 或 system preset 中引用它。不要把完整内容直接塞进主 prompt，而是通过文件注入，保持可编辑和可版本化。

```
agent/
  IDENTITY.md
  memory/
    evolution.md
  tools/
```

### 2. 给身份分层

IDENTITY.md 建议固定四个部分：

- **Core**：不可轻易变更的目标、边界、禁止行为。例如“只在 tools/ 下写入文件”“对外部请求先确认再执行”。
- **Operating**：可调整的工作偏好，如“遇到不确定参数先短暂追问，不要自行虚构”。
- **Memory Pointer**：指向长期记忆、项目笔记、复盘文件，不把大段内容塞进身份文件。
- **Evolution**：追加式记录，保存每次复盘结论、规则调整、版本变化。

示例：

```markdown
# Core
- 目标：维护本地自动化工作流。
- 边界：不得修改 memory/ 下文件。

# Operating
- 遇到模糊指令，先复述理解再执行。

# Memory Pointer
- 长期记忆：memory/long_term.md
- 最近复盘：memory/evolution.md

# Evolution
- 2025-01-12：增加“不得覆盖用户未提交的配置”。
```

### 3. 加载顺序与优先级

身份文件的加载位置很关键。通常我会放在工具说明之前、默认系统规则之后。这样既可以覆盖一部分默认行为，又不会提前消耗太多上下文。若 OpenClaw 支持多段 system 注入，可以用“默认规则 → IDENTITY.md → 工具/插件说明”的顺序加载。

### 4. 让身份进化，而不是频繁改 Core

进化不是每次会话后都改 core。我会在以下触发条件下才更新 Evolution：

- 出现一次明确的执行错误；
- 用户纠正某个行为；
- 任务完成后做简短复盘。

更新时只追加记录，不修改 Core。Core 的修改需要单独评审，因为它是稳定边界。

## 踩坑点

1. **身份文件过长**。如果 IDENTITY.md 超过约 800-1200 tokens，就会开始挤压工具描述和 MCP 指令。控制身份总量，让核心规则短小明确。
2. **与系统提示冲突**。OpenClaw 默认规则里可能有“优先执行工具调用”，而你的 IDENTITY 写“遇到不确定先询问”，会造成 agent 表现矛盾。解决方法是明确优先级，并在测试时用冲突样例验证。
3. **把对话内容直接写进身份**。比如把某次情绪化反馈或一次性偏好变成永久规则，导致身份被污染。长期偏好需要经过筛选，不能自动写入。
4. **跨项目复用同一份身份**。不同项目的边界和工具不同，直接复用会让 agent 带上上一个项目的模式。建议按项目拆分身份文件，公共部分用模板引用。
5. **在身份文件中写敏感信息**。IDENTITY.md 可能被工具读取、日志记录或导出，密钥、token、个人隐私都不该出现在里面。

## 可复用建议

- 使用 Markdown 分区 + YAML front matter，便于解析和自动化更新。
- 把 IDENTITY.md 纳入 git 版本控制，让每次“进化”可回溯。
- 在 agent 的最终输出或自检信息里加上“当前身份版本”，方便快速定位问题。
- 保持 Core 小、Operating 可调、Evolution 追加，这样身份既有稳定性，又能随项目演进。

## 总结

IDENTITY.md 不是另一个更长的 system prompt，而是把“AI 是谁、能做什么、不能做什么”从上下文里抽出来，做成一个可演进、可版本化的工程文件。它解决的不只是人格化问题，更是 OpenClaw 长期运行、多 agent 协作时的行为一致性和可维护性。

如果你的 OpenClaw 实例经常跑偏，不妨先把“身份层”从主 prompt 里剥离，试试用少量、分层、可回溯的 IDENTITY.md 作为锚点。它不会让 AI 变得不可控，反而会让每次变化都有据可查。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/c1a90460d878a81c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/1404ea1ec3e8ed34.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/14fe51e90dce516f.png)

