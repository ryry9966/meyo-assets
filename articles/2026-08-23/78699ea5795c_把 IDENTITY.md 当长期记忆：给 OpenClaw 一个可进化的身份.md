---
title: 把 IDENTITY.md 当长期记忆：给 OpenClaw 一个可进化的身份
feedId: 34366
source: 综合讨论
publishedAt: 2026-08-23
---

# 把 IDENTITY.md 当长期记忆：给 OpenClaw 一个可进化的身份

## 背景

OpenClaw 项目里，Agent 每次会话都需要知道“我是谁、该做什么、边界在哪”。常见做法是在 system prompt、插件配置或项目说明里写一段身份描述。问题是：项目一旦推进，这些描述很快过时。MCP server 从 stdio 迁到 SSE、自动化脚本换了入口、某个插件被停用，身份文件却还停留在旧状态。AI 每次会话学到的东西也没有沉淀，下一次仍然从接近空白开始。

IDENTITY.md 可以看作放在仓库根目录、受 Git 管理的“可进化身份文件”。它不该只是固定 prompt，而是一份可审编、可回滚、可归档的长期记忆索引。

## 问题

1. **固定身份跟不上项目变化**：AI 反复踩同一个坑，因为身份文件里没有记录既往结论。
2. **无约束记忆会变成噪音**：AI 容易把一次偶发失败写成普适规则，或者生成大量泛泛而谈的总结。
3. **跨项目身份冲突**：全局身份和项目身份混在一起，导致不同仓库之间互相污染。

## 做法：最小稳定身份 + 受控经验缓冲

### 1. 把 IDENTITY.md 拆成两层

顶部放稳定身份，底部放动态经验。稳定身份尽量少改，动态经验则用固定字段记录。

```markdown
---
version: 1.3
updated: 2026-03-27
scope: project:openclaw-cn-research
---
## Stable Identity
- 你是 OpenClaw-CN 项目助手，只处理本仓库范围内的自动化与 MCP 配置问题。
- 默认不修改生产配置，除非用户明确确认。

## Experience Buffer (max 15)
- date: 2026-03-27
  context: "MCP server connection timeout"
  rule: "先检查 transport 类型；stdio 与 SSE 的配置字段不同，不能混用"
  status: accepted
```

这样 AI 在会话开始时可以快速恢复关键约束，而不需要重新推导。

### 2. 让 AI 只提 diff，不允许直接写文件

在 OpenClaw 的 post-task 钩子或自动化插件里，让 AI 在会话结束时提议一条更新：

```bash
# .openclaw/hooks/post_task.sh 示意
openclaw identity propose --diff-only > .openclaw/identity_proposal.diff
```

如果当前 OpenClaw 版本没有对应命令，可以用插件调用文件读取 + `git diff` 实现相同效果。关键是：AI 只负责提出修改建议，最终写入必须经过人工确认。

### 3. 人工审编 + Git 归并

IDENTITY.md 纳入 Git 管理。AI 的提议以 diff 或待审清单呈现，人工确认后再提交。删除比添加更重要——很多过时规则如果不清理，会比没有记忆更危险。

### 4. 设置容量与归档

Experience Buffer 建议设置上限，例如 15 条。满额后把旧条目移到 `archive/identity/`，或按主题归并成一条更抽象的规则。不要让身份文件无限增长。

## 踩坑点

- **身份文件膨胀**：AI 会为了“努力”而写大量无信息量总结。只允许记录包含明确触发条件和可执行结论的条目。
- **自我更新固化错误**：一次 SSE 失败后，AI 可能写出“永远不要用 SSE”。要区分“事实约束”和“经验假设”，后者必须人工确认后才能标记为 accepted。
- **格式漂移**：不锁死字段时，AI 会写成自然语言段落，后续解析和检索困难。用 YAML frontmatter 和固定字段最稳。
- **全量注入成本**：如果每次会话都把整个 IDENTITY.md 塞进上下文，大文件会吃掉大量 token。建议只注入稳定区 + 最近 5 条经验，旧经验按需检索。
- **配置事实与身份描述不同步**：AI 更新了 IDENTITY.md 中关于 MCP 的描述，但实际配置文件没有改。应把“身份描述”和“配置事实”分离，配置以真实文件为准，身份只记录决策和边界。

## 可复用建议

- **分层管理**：全局身份放 `~/.openclaw/identity.md`，项目身份放仓库 `IDENTITY.md`，会话内存放临时目录，不要互相替代。
- **固定状态字段**：经验条目使用 `proposed / accepted / rejected / archived` 生命周期，避免模糊状态。
- **非显而易见才写入**：已经写在官方 README 或文档里的内容，不要复制进身份文件。
- **定期清理**：每 1–2 周归并可合并的经验，删除过时规则。身份文件不是日志，是决策索引。

## 总结

IDENTITY.md 的价值不在于给 AI 一个完整人格，而在于提供一层可审编、可回滚、可归档的长期记忆。真正可进化的不是 AI 自动写入的能力，而是“AI 提议 + 人审编 + Git 版本化”的闭环。把身份文件当代码管，比把它当 prompt 更稳定。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/91910f33a858eb5d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/1b1e08bbce24cdc2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/a9c49a5d5f54ae0e.png)

