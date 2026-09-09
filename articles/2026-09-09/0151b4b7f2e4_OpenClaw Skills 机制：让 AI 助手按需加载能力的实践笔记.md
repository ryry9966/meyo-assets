---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力的实践笔记
feedId: 36792
source: 综合讨论
publishedAt: 2026-09-09
---

## 背景

对自建 Agent 来说，能力边界基本由上下文决定。早期常见的做法是把所有指令、工具说明、操作 SOP 全部塞进 system prompt。能力一多，token 成本和噪声同步上涨：模型要在几万 token 里挑出相关的几段，响应质量反而下降。

OpenClaw 的 Skills 机制核心是**渐进式披露**：常驻上下文的只有每个 skill 的 `name` + `description`，正文在命中时才加载，正文里引用的脚本和参考文件再按需读取。三层结构，层层懒加载。

## 问题

实际落地要解决三件事：

1. **触发判断**：description 写不好，该触发时不触发，不该触发时乱入；
2. **上下文预算**：skill 正文过长，等于换个地方堆 token；
3. **与 MCP 的分工**：哪些做成 skill，哪些做成 MCP tool。

## 做法

### 1. 目录结构

一个 skill 就是一个目录，入口是 `SKILL.md`：

```
~/.openclaw/workspace/skills/
└── weekly-report/
    ├── SKILL.md              # YAML frontmatter + 指令正文
    ├── scripts/render.py
    └── references/style.md
```

frontmatter 最少两个字段：

```yaml
---
name: weekly-report
description: 生成周报。当用户要求汇总本周工作、输出周报结构时使用；不用于临时任务清单。
---
```

正文写操作指引：使用时机、步骤、边界（不要做什么）、输出格式。这个格式沿用 AgentSkills 约定，已有的 skill 目录可以直接放进来。

### 2. 写 description 的原则

description 是模型的路由依据。写清两件事：**什么时候用、什么时候不用**。模糊的“帮助处理报告”，不如“用户明确要求生成周报或汇总本周提交记录时使用”。把反例场景也写上，误触发会明显减少。

### 3. 安装与验证

自建 skill 放 `workspace/skills` 即可；社区 skill 走 ClawHub：

```bash
openclaw skills install <name>
openclaw skills list
openclaw skills info <name>
```

装完记得重启 gateway，再用 `skills list` 确认加载。批量启停用 `openclaw.json` 里的 allow/deny 列表控制。

### 4. 分层懒加载

正文控制在一屏内，细节外置：长参数说明放 `references/`，可执行逻辑放 `scripts/`，`SKILL.md` 里只写“需要时读取某文件”。模型会按指引自行读取，不读就不占上下文。

## 踩坑点

- **description 太短太泛**，是“skill 不触发”的最常见原因，优先改这里；
- 正文写成产品文档而不是指令，模型要的是可执行步骤，不是介绍；
- 装完忘了重启 gateway，以为 skill 失效，排查半天；
- `scripts/` 里的脚本没有可执行权限，或解释器路径不对，执行直接报错；
- 把 skill 当 MCP 用：skill 是提示词层面的能力，没有 schema 校验和结构化返回。对接外部 API、需要严格参数约束的场景，老老实实用 MCP。

## 可复用建议

- 先挑 3~5 个高频流程做 skill，别一上来铺几十个；
- 团队内统一模板：适用场景 / 前置条件 / 步骤 / 禁止事项 / 输出格式；
- 用真实对话测触发，迭代 description 比改正文更有效；
- skills 目录进 git，多机同步后配合 workspace 挂载直接生效；
- 分工口径一句话：**MCP 管接口，skill 管流程**。

## 总结

Skills 解决的是“能力多而上下文贵”的矛盾：元数据常驻、正文按需、资源再按需。实践重点不在目录结构，而在 description 的触发精度和正文的指令化程度。建议从一个真实的痛点流程开始，跑通“写 skill → 测触发 → 收敛正文”的闭环，再逐步扩展。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/fc0eb32757105d17.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/91dddbbec613cfbf.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/aa2db385e1732fcd.png)

