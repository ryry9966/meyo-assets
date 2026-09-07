---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 36460
source: 综合讨论
publishedAt: 2026-09-07
---

## 背景

OpenClaw 的 agent workspace 里有一组 Markdown 载体的“人格层”文件：`AGENTS.md` 管行为规则，`SOUL.md` 管性格与底线，`USER.md` 记用户偏好，`MEMORY.md` 记长期记忆。`IDENTITY.md` 是其中最薄、也最容易被忽略的一份——它只回答一个问题：**这只 agent 是谁**。字段很少：`name`、`creature`、`emoji`、`vibe`、`avatar`、`interaction notes`。文件小到十分钟就能填完，但它决定了模型在每次会话启动时“带着什么自我认知入场”。

## 问题

默认装好的 agent 没有身份，在工程上表现为三类麻烦：

1. **语气漂移**：同一个会话里一会儿像客服、一会儿像极客，跨会话更难保持一致，复现问题也无从下手。
2. **多实例混淆**：同时跑三四个 agent 时，日志和通知里分不清“这句是谁说的”。
3. **调优无锚点**：想改风格时没有明确、可 diff 的修改入口，只能往 system prompt 里堆补丁，越堆越乱。

## 做法

1. **初始化**：onboarding 阶段让 agent 自己起草第一版（它会自荐名字、emoji 和 vibe），人只保留否决权。自荐版本往往比人硬编的更自洽。
2. **字段化**：严格保持六个字段，不要自由发挥写长段落。vibe 两三行短语即可。
3. **建立进化回路**：单独准备一个观察笔记，日常使用中遇到“语气不对、该主动没主动、太啰嗦”的片段就记一条；攒够三五条，合并进 `interaction notes`，统一用“**场景 → 期望行为**”格式写，例如：深夜排查故障 → 直接给结论和命令，不寒暄。
4. **git 管理整个 workspace**：每次改身份一个 commit，写清动机；行为异常时能二分回滚。
5. **多 agent 用模板生成**：同一份骨架 + 脚本替换 `name`/`emoji`/`vibe`，保证结构一致，只在内容上区分。

## 踩坑点

- **用户偏好写进了 IDENTITY.md**：那是 `USER.md` 的职责。混进去之后换设备同步、换账号复用，身份就被污染了。
- **vibe 写成小作文**：吃掉上下文预算还稀释权重，长篇性格描述反而让风格不稳定。
- **改完旧会话没生效**：identity 是会话启动时注入的，改完必须开新会话验证，别误判为“改了没用”。
- **昵称风格与正式场景冲突**：工作汇报里被叫昵称很怪。在 interaction notes 里明确“什么场合用什么称呼”。
- **与 AGENTS.md 打架**：身份说“健谈”，规则说“回答不超过三句”，模型行为会抖动。冲突时规则优先，身份让路。

## 可复用建议

- 把身份**当配置而不是文案**：可 diff、可回滚、可评审，改动走 commit。
- 一次只改一个字段，观察几天再决定去留，避免归因混乱。
- interaction notes 控制在十条以内；超了通常说明问题其实出在 SOUL.md 或 AGENTS.md 层面，该迁移就迁移。
- 给每版 identity 打 tag，出问题时能快速定位“哪次修改引入的”。

## 总结

`IDENTITY.md` 的价值不在于“让 AI 更像人”，而在于把风格类问题从玄学变成配置管理：小步修改、git 留痕、定期合并观察笔记，让它随真实使用持续演化。身份稳定的 agent，日志可读、行为可预期，多实例协作时排查成本低得多。如果你的 workspace 里它还是一份空模板，值得花十分钟填一遍，再花两周慢慢养——这大概是人格层文件里投入产出比最高的一份。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/7d5ee68085fba451.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/59a24be0ef0c94cf.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/67220feedb794253.png)

