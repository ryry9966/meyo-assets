---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份，而非一次性提示词
feedId: 34349
source: 综合讨论
publishedAt: 2026-08-23
---

## 背景：Agent 的长期记忆，不应该只靠聊天记录

在 OpenClaw 里跑 Agent 时，最常见的问题是：每次开新会话，它就像换了个人。你上次告诉它的“不要自动执行删除”“输出优先用中文”“先读 README 再动手”，它全忘了。单靠聊天记录或 memory 插件，能记住碎片，但很难维持一致的行为边界。

IDENTITY.md 的思路很简单：给 OpenClaw Agent 一个可读、可版本控制、可进化的身份文件。它不负责存大量事实，只负责定义“我是谁、我该做什么、我默认怎么做、我绝对不做什么”。

## 问题：身份散落各处，无法维护

通常我们会把身份写在 system prompt 里，或者塞进 memory 插件。长期运行后会暴露几个问题：

- 规则冲突：system prompt 说“谨慎操作”，memory 里又记着“用户喜欢激进自动化”。
- 无法审阅：改动没有历史，你不知道 Agent 为什么变成现在这样。
- 容易漂移：每次让 Agent“记住一点”，它可能把临时偏好写成永久规则。
- 没有安全边界：身份文件里混着 token、路径、个人偏好，一旦被共享或输出，风险很高。

IDENTITY.md 不是要替代这些，而是提供一个稳定的锚点。

## 做法：把身份文件纳入配置与版本管理

### 1. 确定文件位置

建议放在项目级目录，而不是全局唯一文件：

```text
project/
├── .openclaw/
│   ├── IDENTITY.md
│   ├── NOTES.md
│   └── SKILLS.md
```

如果只是个人全局助手，可以放 `~/.openclaw/IDENTITY.md`。项目级的好处是：不同项目可以有不同边界，避免“个人身份”污染“工作自动化”。

### 2. 写一个最小模板

不要一开始就写几千字。最小可用模板如下：

```markdown
# IDENTITY

## 身份
- 名称：openclaw-local
- 角色：个人自动化助手

## 职责边界
- 负责：文件整理、脚本执行、信息检索
- 不负责：未经确认的删除、对外发布

## 默认工作流
1. 先读 README 或相关说明
2. 列出将要执行的操作
3. 危险操作前等待用户确认

## 偏好
- 输出使用中文
- 命令优先使用跨平台语法

## 禁忌
- 不读取 .env 中的密钥并写入日志
- 不修改本文件，除非用户明确要求

## 变更策略
- 手动编辑为主
- Agent 可提出修改建议，但必须等待合并
```

这个模板的关键是“可执行”，每一条都能在行为上验证，而不是“友好、智能”这种空话。

### 3. 在 OpenClaw 引导中引用

OpenClaw 不会自动读 IDENTITY.md，需要你在启动配置或系统引导中显式引用。例如：

```text
Read ~/.openclaw/IDENTITY.md and follow it as your persistent operating rules.
```

或者把 IDENTITY.md 内容作为 system prompt 的一部分注入。确保每次会话开始都会读，而不是只在第一次读。

### 4. 更新机制：手动为主，AI 建议为辅

不建议直接给 Agent 写入 IDENTITY.md 的权限。更好的做法是：

- 你手动编辑 IDENTITY.md，然后提交 git。
- Agent 发现问题时，输出一个 diff 或建议段，你 review 后合并。
- 如果一定要让 Agent 修改，至少让它写到 `NOTES.md`，而不是直接改 `IDENTITY.md`。

这样身份文件才是“可进化的”，而不是“被随便改坏的”。

### 5. 纳入 git 版本管理

在 `.openclaw/` 目录下执行：

```bash
git init
git add IDENTITY.md
git commit -m "init identity"
```

以后每次修改都有记录。遇到行为异常，可以先看身份文件最近改了什么。

## 踩坑点

1. **把敏感信息写进去**：IDENTITY.md 经常被 Agent 全文读取，甚至可能在日志、调试输出中出现。Token、密码、内网路径不要放。

2. **文件过载**：超过 300-500 行后，模型注意力会稀释，重要规则被忽视。把动态内容拆到 NOTES.md。

3. **让 Agent 自动改身份**：Agent 在某次任务后可能把“用户喜欢激进自动化”写进 IDENTITY.md，之后每次都想跳过确认。身份文件的可写权限必须收紧。

4. **编码问题**：Windows 下用 UTF-8 保存，避免中文注释变成乱码。路径最好用正斜杠或绝对路径。

5. **与 memory 插件冲突**：如果 memory 里也存了规则，可能出现两套说法。建议 IDENTITY.md 只放稳定规则，memory 只放短期事实。

6. **提示注入污染**：如果 Agent 会读网页或邮件，攻击者可能诱导它“更新身份文件”。所以不要让 Agent 直接写身份文件。

## 可复用建议

- **稳定与动态分离**：`IDENTITY.md` 放长期身份，`NOTES.md` 放临时上下文，`SKILLS.md` 放能力清单。
- **只读核心 + 可写边栏**：核心身份文件只读，Agent 只能写 NOTES 或提出 diff。
- **变更像代码一样 review**：在 git 仓库中开分支，Agent 的修改放在 branch，你确认后再 merge。
- **定期 review**：每周或每月检查一次身份文件是否还符合实际任务。过时的规则比没有规则更危险。
- **改完做行为测试**：改完身份文件后，跑几个固定场景：危险操作是否询问、输出语言是否正确、禁用命令是否被触发。

## 总结

IDENTITY.md 不是给 Agent 写一段“人格设定”，而是一个工程化的身份锚点。它让 Agent 的行为有连续性，让身份变更可审阅、可回滚。可进化的前提是约束：核心稳定、变更留痕、权限最小。

如果你的 OpenClaw Agent 还在靠聊天记录维持人设，不妨先建一个 20 行的 IDENTITY.md，然后把它交给 git。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/ac24f260a8413465.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/ed0fcacf11aecb24.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/ea89515df95e886e.png)

