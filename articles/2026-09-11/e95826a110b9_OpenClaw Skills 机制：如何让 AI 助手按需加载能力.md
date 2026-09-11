---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 37053
source: 综合讨论
publishedAt: 2026-09-11
---

## 背景

给 Agent 接能力的常见做法是把所有工具一口气挂上：MCP server 全开、插件全装、系统提示词里塞满说明。能力多到一定程度，模型开始"选错工具"，上下文成本也跟着涨。OpenClaw 的 Skills 机制走的是另一条路：技能平时在提示词里只占一行"名字 + 一句描述"，当任务真正匹配时，完整的 SKILL.md 正文才会被加载进上下文——典型的渐进式披露（progressive disclosure）。

## 问题

我们在做一个内部数据助手时，最初挂了十几个 MCP 工具，外加一份两万字的说明文档，结果：

- 系统提示词常驻 8k+ tokens，每轮对话都在为用不到的能力付费；
- 模型经常把"查数据库"和"跑 Python 脚本"两个工具混用；
- 环境里没配 API key 的工具照样暴露给模型，一调用就报错。

## 做法

Skills 的本质是用文件系统组织能力包，最小单元是一个目录加一个 SKILL.md：

```markdown
---
name: csv-report
description: Generate weekly CSV summary reports from local data.
  Use when the user asks for data exports or weekly summaries.
metadata:
  requires:
    bins: ["python3"]
---

# CSV Report

1. 读取 ./data/ 下的原始文件
2. 执行 scripts/build_report.py，参数为目标周
3. 输出到 ./reports/，命名 YYYY-Www.csv

字段映射细节见 references/schema.md，按需读取。
```

三个关键点：

1. **description 是路由键。** 模型只靠这一句决定是否加载正文，所以要用第三人称写清"什么场景用"，而不是堆关键词。
2. **正文保持轻。** 长文档放到 references/ 之类的附属文件，SKILL.md 里只写路径引用，用到才读。
3. **放对位置。** 工作区 skills 目录（如 `~/.openclaw/workspace/skills`）优先级最高，可覆盖内置同名技能，改动即时生效，无需重启。

## 踩坑点

- **description 写成关键词堆砌**：技能要么永远不触发，要么乱触发。写成 "Use when the user asks for X" 这种句式后，触发率明显更稳。
- **和内置技能重名**：重名会覆盖内置版本。排查问题时容易误判为官方逻辑问题，其实是自己的同名文件在生效。
- **技能数量失控**：五六十个技能的"索引"本身也会撑大提示词。建议先合并同类，再做加法。
- **门控没配**：环境变量或二进制缺失时技能会"半可用"，在 `metadata.requires` 里提前声明 bins/env，Agent 会直接跳过。
- **写死绝对路径**：换台机器就断。脚本和文档统一用相对路径，跟着技能目录走。
- **密钥写进 SKILL.md**：这是会被注入上下文的，密钥一律走环境变量加门控。

## 可复用建议

- 一个技能一个能力，粒度宁粗勿细；
- 把跑通的内置技能当结构模板来抄；
- 技能目录进 git，多设备同步，改动可回溯；
- 定期用 `openclaw skills list` 核对实际加载了哪些技能，别凭感觉。

## 总结

Skills 不是新魔法，本质就是"提示词侧的懒加载 + 文件化的能力打包"。真正决定体验的是两件事：description 写得准不准，以及你有没有克制地控制技能总量。把路由交给描述、把细节交给文件之后，上下文省下来了，工具选择的准确率反而是我们上线 Skills 后提升最明显的指标。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/be17d5623a857568.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/7020029c7b84ab4b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/1fa1a725f819f08a.png)

