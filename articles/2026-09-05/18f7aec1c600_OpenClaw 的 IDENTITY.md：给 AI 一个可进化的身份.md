---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 36204
source: 综合讨论
publishedAt: 2026-09-05
---

## 背景

跑一个长期在线的 Agent，最先遇到的问题往往不是能力，而是"人格不稳定"：今天称呼得体，明天突然变油腻；写周报是一种口吻，回消息又是另一种。OpenClaw 把身份定义从散落的 prompt 里抽出来，放进 workspace 根目录的 `IDENTITY.md`，每次会话启动时注入 system prompt。这个设计很小，但值得认真对待——它是每轮对话都会被读到的唯一身份源。

## 问题

没有 IDENTITY.md 时，常见状态是：

- 身份指令散在 channel 配置、skill 描述、甚至某条历史消息里；
- 换个入口（IM、WebChat、CLI）表现不一致；
- 想改语气只能翻配置，改完不知道影响面，也没法回滚。

本质上是身份没有 single source of truth，也没有版本化。

## 做法

我的 workspace 结构大致是：

```
workspace/
├── IDENTITY.md   # 是谁：名字、形象、语气、边界
├── SOUL.md       # 行为准则：做事方式、价值观
├── USER.md       # 用户是谁：称呼、偏好、时区
└── MEMORY.md     # 长期记忆，由 Agent 自己写
```

IDENTITY.md 我只写四类信息：

1. **名字与形象**：一个名字加一个 emoji，足够模型自我指代即可；
2. **语气（vibe）**：用行为描述代替形容词，比如"回复默认两三句，先给结论再给一句理由"，而不是"简洁友好"；
3. **边界**：哪些事要确认再做（花钱、删文件、对外发消息），哪些可以直接干；
4. **默认语言与时区**：中文回复、Asia/Shanghai，避免每次靠猜。

## 踩坑点

- **写太长**。超过一屏，关键指令被稀释，模型反而开始自由发挥。我压到 30 行以内后稳定性明显提升。
- **放错文件**。工具用法归 TOOLS.md，用户偏好归 USER.md，IDENTITY.md 只管"你是谁"。把任务清单塞进去，等于污染身份。
- **形容词无效**。"要专业""要有温度"基本不改变行为，换成可观察的规则（"不用感叹号""先复述需求再动手"）才有用。
- **改完不重开会话**。文件在会话启动时读取，热改不生效，验证前记得开新会话。
- **整份抄别人的**。身份是长出来的，不是配出来的。别人的 vibe 直接套上，一两天就会和你实际的使用方式打架。

## 可复用建议

把 IDENTITY.md 当配置代码管理：

- 放进 git，一次只改一行，commit message 写清动机（如"回复过啰嗦，限制默认长度"）；
- 每周小复盘一次：这周 Agent 哪些行为让你不爽？能写成行为规则的，就追加或修改；
- 出问题先 diff 身份文件。很多"突然变笨"，其实是上次改 vibe 改坏的，回滚即可；
- IDENTITY.md 求稳，MEMORY.md 求活，两个别混。身份文件一个月改三次以内，是健康的频率。

## 总结

IDENTITY.md 的价值不在"个性化"这个噱头，而在工程上：一个可版本化、可回滚、可 diff 的身份单一来源。Agent 的能力和记忆会自己生长，身份这层底座需要你亲手维护——写少、写具体、小步改。当你发现自己开始频繁微调 Agent 的语气时，往往不是它的问题，是你的 IDENTITY.md 该迭代了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/506f4b434064eca4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/f61ad0f8ba811b6d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/17d0fdda604d45df.png)

