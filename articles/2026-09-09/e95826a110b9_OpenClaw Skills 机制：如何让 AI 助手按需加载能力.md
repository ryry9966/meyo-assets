---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 36672
source: 综合讨论
publishedAt: 2026-09-09
---

## 背景

过去半年我们在内部 Agent 项目里持续堆工具：代码检索、日志查询、工单操作、内部文档搜索，全部注册成 function calling 的 tools。结果是 context 里常驻几十份工具描述，token 开销大，模型在相似工具之间选错的概率也明显上升。

OpenClaw 的 Skills 机制解决的就是这个问题：把“能力”从“始终在线”改成“按需加载”。

## 问题本质

传统工具注册是全量注入，所有 tool schema 每次请求都进 context，带来三个问题：

1. token 成本随工具数线性增长；
2. 工具越多，模型混淆概率越高；
3. 复杂能力（比如一条完整的发布流程）靠单个 function 描述不清。

Skills 的思路是把能力与触发时机绑定：平时只放一份极简元信息，命中条件后才加载完整内容。

## Skills 的结构

一个 Skill 本质是一个目录，核心是带 frontmatter 的描述文件：

```text
skills/
  release-flow/
    SKILL.md        # name + description + 触发条件 + 操作指引
    scripts/
      deploy.sh     # 可选：配套脚本
    references/
      runbook.md    # 可选：深度参考资料
```

关键在于三层渐进式披露（progressive disclosure）：

- **第一层**：启动时只注入每个 Skill 的 name + description，几十 token；
- **第二层**：模型判断任务匹配某 Skill，主动读取 SKILL.md 正文；
- **第三层**：正文再引用 references/ 下的长文档，按需进一步加载。

常驻开销从 O(全部文档) 降为 O(技能数 × 描述长度)。

## 实操步骤

1. 盘点现有工具，优先把“低频 + 高复杂度”的能力改造成 Skill；
2. 写 SKILL.md：name 用小写连字符，description 一句话写清“什么时候该用我”，这是触发准确率的关键；
3. 正文用 checklist 和命令片段组织，控制在 500 行以内，长内容拆到 references；
4. 确定性步骤（构建、部署）放 scripts/，让模型调脚本而不是自由发挥；
5. 灰度上线：先让测试 Agent 只挂这一个 Skill 跑真实任务，观察触发日志。

## 踩坑点

- **description 含糊**是最常见失败原因。写“处理发布相关任务”不如写“代码合入 main 且用户要求部署到生产时使用”。
- **数量失控**。超过 30 个后，光元信息就会挤占 context，需要按业务域分组或加路由层。
- **硬编码路径和环境变量**，换台机器就失效，应统一从注入配置读取。
- **Skill 互相引用未声明**，运行时才发现缺依赖，加载阶段应做一次依赖校验。

## 可复用建议

- 粒度标准：一个 Skill 对应“一个用户意图”，不要拆成“一个函数一个 Skill”；
- description 按检索索引来写：“触发条件 + 输入 + 输出”三段式；
- 脚本与文档分离，脚本要幂等、支持 dry-run；
- 记录每个 Skill 的触发命中率，长期低于 10% 的考虑合并或下线。

## 总结

Skills 机制的价值不在“多一种插件格式”，而在于把 Agent 的 context 从静态全量改成动态按需。我们实测把 12 个低频工具收敛成 5 个 Skill 后，单次请求常驻 token 降了约 40%，工具误选率也明显下降。核心就一句话：让模型在任何时刻只看见它当前需要的能力。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/26e75f095b597b64.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/190212352a1db013.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/d4981d735e0136fb.png)

