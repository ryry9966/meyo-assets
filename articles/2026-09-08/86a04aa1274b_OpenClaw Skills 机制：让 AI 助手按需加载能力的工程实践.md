---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力的工程实践
feedId: 36568
source: 综合讨论
publishedAt: 2026-09-08
---

## 背景

在 OpenClaw 里扩展助手能力，通常有两条路：MCP / 工具解决"能调用什么"，Skills 解决"知道怎么做事"。Skill 本质上是一个文件夹，核心是一个带 frontmatter 的 `SKILL.md`，外加可选的脚本和参考文件，放在工作区的 `skills/` 目录下，Agent 运行时按需读取。

它解决的不是"功能不够"，而是"上下文不够"。

## 问题

早期我把所有操作规范都堆进系统提示词：部署流程、日志排查、发布检查单……写到几千 token 之后出现三个症状：模型注意力被稀释，简单指令也开始出错；每次请求成本固定上涨；改一条规则要重启会话。MCP 侧同理，几十个工具的 schema 常驻上下文后，选错工具的概率明显上升。

Skills 的核心设计是**渐进式披露**：启动时只把每个 Skill 的 `name` + `description` 注入上下文，正文和附属文件在模型判断相关时才加载。一句话概括：**目录常驻，正文按需**。

## 做法与步骤

1. 建目录：`skills/pdf-report/SKILL.md`
2. 写 frontmatter 和正文：

```markdown
---
name: pdf-report
description: 当用户要求把数据整理成 PDF 周报时使用。覆盖模板选择、图表生成与导出。
---

# PDF 周报生成

1. 读取 data/weekly.csv
2. 调用 scripts/render.py 渲染模板
3. 输出到 output/

图表参数细节见 reference/chart-spec.md
```

3. 关键认知：**description 是路由信号**。模型靠它决定是否加载正文，所以要写"什么时候用"，而不是"这是什么"。
4. 重载会话后验证：抛一个应当触发的问题，观察回复是否引用了 Skill 里的步骤。

这样形成三档加载：元数据常驻 → 正文触发时读 → 链接文件用到才读。

## 踩坑点

- **description 写成名词解释**，永远不会被触发。改成"当用户要求 X 时使用"。
- **职责太大**。一个"运维 Skill"塞了部署、告警、巡检，路由必然糊，拆开。
- **语义重叠**。两个 Skill 的 description 高度相似时模型随机选一个。新增前先全局搜一遍已有 description。
- **密钥写进 SKILL.md**。它是纯文本，会进上下文也会进 git。密钥放环境变量，Skill 里只写引用方式。
- **附属脚本没有可执行权限**，触发后报错，模型随即开始自由发挥——这类失败最难排查。
- **数量失控**。几十个 Skill 的元数据本身也是负担，低频的直接下线。

## 可复用建议

- 一个 Skill 只做一件事，命名用动词短语；
- 正文控制在两百行内，细节外移到链接文件；
- Skill 目录进 git，改动走 PR，当代码 review；
- 维护一组探针 prompt，每次改动后跑一遍触发验证；
- 能用脚本确定的步骤写成脚本，Markdown 只负责编排。

## 总结

Skills 不是又一个插件系统，而是把"上下文经济学"落成了工程约定：常驻的只有索引，能力本体按需加载。实践中最大的体会是——写好一条 description，比堆十个功能更重要。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/352507d5a3b0398d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/8c0591cee811ca62.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/5f478c2e8ada2166.png)

