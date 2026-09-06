---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 36378
source: 综合讨论
publishedAt: 2026-09-07
---

## 背景

OpenClaw 的 Agent 行为并不靠硬编码，而是靠 workspace 里的几份 Markdown 文件组装出来的：`AGENTS.md` 定规则，`SOUL.md` 定性格与价值观，`IDENTITY.md` 定"我是谁"。每次会话，这些文件会被注入 system prompt，Agent 读到什么，就是什么。

这套设计最容易被忽视的就是 `IDENTITY.md`。很多人装完就忘了，Agent 顶着默认身份跑了半年。但用下来你会发现：身份文件是成本最低、收益最明显的一个配置点。

## 问题

没有明确身份文件的 Agent 有三个典型毛病：

1. **跨渠道人设漂移**。Telegram 里叫 A、Discord 里自我介绍是 B，用户困惑，日志难排查。
2. **名字没存在感**。你叫它"小助手"，它回复里从不自称，长期交互缺少一致性锚点。
3. **身份和行为规则混写**。把"回复要简短"这类规则塞进身份文件，改一处影响全局，迭代困难。

## 做法

workspace 默认在 `~/.openclaw/workspace`（或你配置的 `OPENCLAW_WORKSPACE` 下），直接编辑 `IDENTITY.md`：

```markdown
# IDENTITY.md

- **Name:** Marco（让 Agent 明确自称）
- **Creature:** 一只戴眼镜的机械章鱼
- **Emoji:** 🐙
- **Avatar:** 🐙（emoji 即初始头像）
```

四个字段各司其职：Name 管自称，Creature 管形象设定，Emoji/Avatar 管视觉锚点。保存后新会话即生效，无需重启 gateway。

建议的迭代流程：

1. 先只填 Name + Emoji，观察一周日常对话；
2. 确认自称稳定后，再补 Creature，让回答风格有据可依；
3. 把整个 workspace `git init`，每次身份调整提交一次，留一条可回滚的进化记录；
4. 多 Agent 场景下，一个 Agent 一个 workspace，身份文件互不污染。

## 踩坑点

- **IDENTITY ≠ SOUL**。身份是静态事实（叫什么、是什么），性格、语气、底线写进 `SOUL.md`。混着写之后，改人设会误伤行为规则。
- **长会话有缓存感**。改完身份文件，旧会话上下文里还是旧人设，开新会话再验证。
- **别超载**。有人把十几条行为指令堆进 IDENTITY.md，结果注入变长、优先级混乱。身份文件保持在个位数行。
- **Avatar 别填路径就完事**。如果配了自定义图片，确认渠道端（如 Telegram）真的会渲染，否则 emoji 反而是最稳的方案。

## 可复用建议

- 把身份当 **config-as-code**：进 git、写 commit message（如 `identity: rename to Marco`），身份演进历史一目了然。
- 身份文件**只做减法不做加法**：字段越少，模型遵循越稳定。
- 配合 `USER.md` 区分"我是谁"和"你是谁"，两份文件不要互相引用。
- 团队场景可以把 IDENTITY.md 做成模板仓库，新 Agent 初始化时 fork 一份再微调。

## 总结

`IDENTITY.md` 看起来只是几行配置，实际上它给了 Agent 一个稳定、可版本化、可进化的自我。先让"我是谁"清晰，再谈"我该怎么做"，这个顺序在 OpenClaw 的文件体系里是成立的。花十分钟填好这四行，回报是长期交互里的一致性——这是我在自己 instance 上验证过的、性价比最高的一处调优。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/ac9d073925cbf862.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/875d04bc3cf6f54f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/edc703fa1117f6d4.png)

