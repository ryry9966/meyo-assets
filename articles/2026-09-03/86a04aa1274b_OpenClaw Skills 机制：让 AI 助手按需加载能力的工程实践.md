---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力的工程实践
feedId: 35910
source: 综合讨论
publishedAt: 2026-09-03
---

## 背景

用 OpenClaw 搭助手，最直观的膨胀来自上下文：MCP 工具说明、自定义流程、操作手册，全量塞进 system prompt。能力一多，静态开销轻松上万 token，模型注意力被稀释，反而在相似工具之间摇摆。Skills 机制解决的就是这个问题——核心是**渐进式披露**：常驻上下文的只有每个 skill 的 name 和 description，正文只在模型判定相关时才注入。

## 问题

我的助手接了十几个 MCP server，外加部署检查、数据导出等自定义流程。全量注入时静态上下文约 2 万 token，代价有两个：一是慢和贵；二是模型经常"背错课文"——明明该走部署检查清单，却引用了数据导出里的步骤。

## 做法

1. **建目录**：在 workspace 下建 `skills/<skill-name>/SKILL.md`，这是最小可用单元。
2. **写 frontmatter**：name 唯一，description 写"什么时候用"，不是"这是什么"。
3. **正文保持精炼**：SKILL.md 控制在两百行以内，细节拆到 `references/*.md`，让模型按需再读。
4. **确定性操作写成脚本**：放 `scripts/` 下，比让模型"照步骤执行"可靠得多。
5. **验证**：用 `/skills` 确认加载状态，再问一个触发性问题，观察它是否走到对应 skill。

```markdown
---
name: deploy-check
description: 当用户要求发布、上线或检查部署状态时使用，含发布前清单与回滚步骤。
---
# 部署检查
1. 确认 CI 通过
2. 执行 scripts/precheck.sh
参数细节见 references/api-notes.md
```

## 踩坑点

- **description 写成名词解释**："这是用于部署的工具"基本不会触发。要写成触发条件："当用户要求发布/回滚时使用"。
- **把所有细节堆进 SKILL.md**：等于换了个地方塞上下文，渐进式披露形同虚设。
- **同名冲突**：自定义 skill 会覆盖内置 skill，行为悄悄变了而不自知，起名前先查一下。
- **脚本相对路径**：工作目录和预期不一致时静默失败，脚本内部用绝对路径，或在正文里显式说明执行目录。
- **skill 数量失控**：几十个 description 本身也是开销，低频 skill 该合并就合并。
- **与 MCP 重复维护**：MCP 管"接口"，skill 管"流程知识"，同一份说明写两遍迟早打架。

## 可复用建议

- 把 description 当 prompt 迭代：不触发就重写，误触发就加"仅当……时使用"的限定。
- 记住三层结构：SKILL.md（索引）→ references（细节）→ scripts（确定性执行），粒度自上而下递增。
- skill 进 git 单独管理，description 的每次修改单独提交，出问题能快速回滚。
- 排障时直接问 agent："你刚才用了哪个 skill，为什么？"这往往比翻日志更快定位路由问题。

## 总结

Skills 的价值不在"多"，而在"准"。它把上下文从静态清单变成按需检索的目录，省 token 只是表象，真正的收益是路由准确率。实践中我的体会是：花在打磨 description 上的时间，远比堆 skill 数量更有回报。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/7a9b682745844701.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/71066fc390f6aac2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/6cfdf48b6727e531.png)

