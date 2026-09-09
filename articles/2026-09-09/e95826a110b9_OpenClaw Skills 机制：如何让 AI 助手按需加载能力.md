---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 36737
source: 综合讨论
publishedAt: 2026-09-09
---

## 背景

给 Agent 扩能力，常见两条路：MCP 提供标准化工具接口，系统提示词注入过程性知识。前者解决"能调什么"，但接口之外还有大量"怎么用"的知识——内部部署流程、排障手册、固定操作套路。这类内容过去只能整段塞进 system prompt。

## 问题

我们早期把二十来份操作手册全文注入系统提示词，每次会话固定多出几万 token，而单次任务实际只会用到其中一份。更麻烦的是，大段无关指令会稀释模型注意力，该遵循的规范反而更容易被略过。能力全量加载，成本和精度两头受损。

## Skills 的做法：两级加载

OpenClaw 的 Skills 用的是渐进式披露（progressive disclosure）：

1. 每个 skill 是一个目录，核心是一个 `SKILL.md`。frontmatter 里的 `name` 和 `description` 常驻系统提示词，开销只有一两行；
2. 正文（具体步骤、脚本说明、注意事项）默认不进上下文，模型判断当前任务命中某个 description 时，才去读全文。

落地步骤：

1. 在 workspace 下建 `skills/<name>/SKILL.md`；
2. frontmatter 写 `name` 和 `description`；
3. 正文写可执行的操作步骤，需要脚本时把脚本放在同目录，正文写清调用方式和路径；
4. 重载后用一个典型问法验证触发。

关键认知：**description 是路由键，不是简介**。它决定 skill 会不会被加载。建议按"用于什么场景 + 什么信号触发"来写，把用户可能说的关键词埋进去。比如"用于发布服务。当用户要求上线、发版、回滚、查发布状态时使用"，远好于"处理发布相关任务"。

## 踩坑点

- **description 太泛或太窄**：太泛会乱触发，太窄永远不触发。上线后看会话日志校准，比凭感觉改快得多。
- **多个 skill 描述重叠**：模型会加载错的那份，然后按错误手册执行。用互斥关键词切分边界。
- **正文过长**：触发瞬间吃掉大量上下文。主流程和细节拆开，正文只留主干，细节放同目录的引用文件，让模型按需再读。
- **把 skill 当 MCP 用**：skill 适合沉淀过程性知识，MCP 适合暴露标准化接口。在 skill 里手写一堆 API 参数细节是反模式，能接 MCP 的别手写。
- **路径问题**：脚本用相对 workspace 的路径，注意 agent 实际工作目录，否则会出现触发成功但执行失败。

## 可复用建议

- 把 skills 当内部文档维护：进 git、走 review，谁踩了坑谁补一段，知识不会只留在某个人脑子里。
- 高频组合操作优先沉淀成 skill。同一个流程口头描述十次，不如写成一份 skill 稳定。
- 定期审计：哪些 skill 从未被触发（描述写偏了）、哪些频繁误触发（边界不清），各处理一批。
- 和 MCP 分工：MCP 给工具，Skills 给方法，一个 skill 串起几个 MCP 工具，是最舒服的组合。

## 总结

Skills 的本质是把 prompt 工程变成知识管理：常驻上下文的只有索引，正文按需加载。token 成本降下来，指令遵循精度反而上去，而且整套东西可版本化、可审计。如果你的 agent 已经接了 MCP，下一步值得做的就是把团队里重复出现的过程性知识沉成 skills——工具给手，方法给脑。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/4abb55c26c9060b7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/7a566581d28cd4b1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/ba3054943f792f22.png)

