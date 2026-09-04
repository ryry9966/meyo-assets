---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 36136
source: 综合讨论
publishedAt: 2026-09-05
---

## 背景

在 OpenClaw 里，agent 的行为不完全是代码决定的。每次会话启动时，workspace 下的一组 markdown 文件会被注入上下文，`IDENTITY.md` 是其中最“人格化”的一份：它定义这个 agent 是谁、叫什么、用什么口吻说话。它不是藏在黑盒里的 system prompt，而是你手里的一个普通文件——可以 git 管理、可以 diff、可以回滚。这意味着“agent 是谁”这件事，可以按工程化的方式演进。

## 问题

不维护身份文件时，常见三个痛点：

- **行为漂移**：没有固定身份，agent 的口吻和边界每次会话都不一样，今天克制明天话痨。
- **调教成果丢失**：你在对话里纠正过十次“别滥用 emoji”，换个新会话又原样复发。
- **不可审计**：设定散落在提示词里，没法 review，改坏了也不知道改了什么。

## 做法

1. **放对位置**：workspace 根目录建 `IDENTITY.md`。
2. **写该写的**：名字、角色定位、语气基线、语言偏好（如“默认中文，术语保留英文”）、边界（如“不确定就说不确定，不要编”）。
3. **保持短**：每个会话都消耗 token，建议控制在 30–60 行，一屏以内。
4. **git 管理**：agent 表现不符预期、你手工纠正后，把结论沉淀成一条规则，commit 进去，附一句变更原因。
5. **小步迭代**：观察两三个会话再改下一版，一次别动太多，否则无法归因。

一个最小骨架：

```markdown
# Identity
- Name: 
- Role: 
- Tone: 
- Language: 
- Boundaries: 
```

## 踩坑点

- **写成小作文**：身份文件不是简历。越长关键指令权重越稀释，“不要编造”这种硬边界会被淹没在一堆形容词里。
- **职责混装**：“我是谁”（IDENTITY.md）、“用户是谁”（USER.md）、“行为准则”（SOUL.md）分开放。把某个用户的偏好写进身份文件，换个使用者就废了。
- **和系统提示打架**：system prompt 和 workspace 文件同时改口吻，agent 会来回横跳。先确认层级，迭代只在 workspace 层做。
- **期望热生效**：改动通常在下一个会话生效，长会话和缓存的场景不会自动加载，验证前先开新会话。
- **整份抄别人的**：覆盖一份现成的 IDENTITY.md，得到的是别人的 agent。这个文件的价值在迭代历史，不在初版。

## 可复用建议

- 把它当 config-as-code：有 changelog、有 review、能回滚。
- 每个迭代周期做一次“蒸馏”：翻聊天记录，把重复纠正超过两次的点写进文件——这是身份进化最可靠的来源。
- 多 agent 场景下，每个 agent 独立 workspace、独立身份文件；共性部分（如安全边界）抽成共享片段分别引用。
- 用 diff 而不是重写：小步修改，才能定位是哪一条规则引起了行为变化。

## 总结

IDENTITY.md 的核心价值不是“调教出一个好人格”，而是把“agent 是谁”从隐性的提示词，变成显性、可版本化、可进化的资产。它很小，但它是你所有调教工作的沉淀层。建议今天就 `git init`，给身份一个历史。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/4bc65d83dc2ae7f9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/ecf23e3f2bb71ff4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/c29c08cff7140e78.png)

