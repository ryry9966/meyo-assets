---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 36954
source: 综合讨论
publishedAt: 2026-09-11
---

## 背景

Agent 用得越久，能力清单越长，system prompt 就越臃肿。早期常见做法是把所有工具说明、操作规范一次性塞进上下文，结果是 token 占用高、注意力被稀释，真正常用的指令反而容易被淹没。OpenClaw 的 Skills 机制给出的是另一种思路：把能力拆成独立的技能包，会话启动时只注入几十字的元信息，模型判断当前任务命中后，才加载完整说明。

## 机制：三级渐进加载

OpenClaw 的 skill 本质是一个目录，核心是一个 `SKILL.md` 文件，加载分三级：

1. **元信息层**：启动时只有 name 和 description 进入上下文，成本几乎可以忽略；
2. **正文层**：当用户请求与某个 description 匹配时，agent 主动读取该 skill 的 SKILL.md 全文；
3. **资源层**：正文中引用的脚本、模板、参考文档，只有在执行时才被打开。

这就是渐进式披露（progressive disclosure）——上下文里永远只放"当前需要的那一层"。

一个最小可用的 skill 长这样：

```
skills/log-analyzer/
├── SKILL.md
├── scripts/scan.py
└── references/patterns.md
```

```markdown
---
name: log-analyzer
description: 分析服务日志、定位错误模式。当用户要求排查报错、
统计错误频率或提取堆栈时使用。
---

# 日志分析
1. 优先运行 scripts/scan.py，不要手写正则
2. 输出超过 200 行时，写入文件后只汇报摘要
3. 常见报错模式见 references/patterns.md
```

## 实操步骤

1. 在 workspace 的 `skills/` 目录下新建技能文件夹，放入带 frontmatter 的 SKILL.md；
2. description 用"用户会怎么提需求"的口吻写触发条件，而不是罗列功能；
3. 正文控制在几十行内，长细节、大表格全部外置到 `references/`；
4. 固定操作写成脚本放进 `scripts/`，让 skill 指向脚本而不是复述步骤；
5. 重启会话后确认 skill 已被识别，再拿几个真实请求验证命中情况。

## 踩坑点

- **description 写成功能说明书**。写"支持日志解析、正则提取、统计报表"，模型很难把它和"帮我看看昨天为什么报错 500"关联起来。要写触发场景，不是写能力清单。
- **把细节全塞进正文**。SKILL.md 写了三百行，等于变相回到全量注入，渐进加载就失效了。
- **脚本路径写死、没有可执行权限**。skill 换台机器跑不起来，多数是这两个原因。
- **多个 skill 的 description 语义重叠**，导致误触发或互相抢占。一个 skill 只做一件事，边界模糊时宁可拆开。
- **改完不重载会话**，以为没生效，其实跑的还是旧上下文。

## 可复用建议

- description 是整个机制里杠杆最大的一行字，值得反复打磨：包含"做什么 + 什么时候用"。
- 把 skill 当成"给 agent 的 README"来维护，而不是 prompt 仓库。
- 定期审计：让 agent 汇报各 skill 的命中频率，长期不命中的要么改描述，要么删掉。
- 团队协作时，skill 目录直接进 git，走 review 流程管质量，比口头约定可靠。

## 总结

Skills 机制没有黑魔法，它只是把 prompt 工程文件化、分层化、按需化。真正的难点不在机制本身，而在描述词的质量和职责边界的划分。建议先用两三个高频场景练手，把 description 打磨到位，再逐步扩充——比一次写几十个 skill 再回来返工划算得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/000c16eb63e63148.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/6314c9afd373589e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/e162b61ede902ae2.png)

