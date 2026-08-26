---
title: OpenClaw Skills 机制：从全量注入到按需加载的工程化实践
feedId: 34827
source: 综合讨论
publishedAt: 2026-08-26
---

## 背景

OpenClaw 的 Agent 运行时通常会挂载不少工具、插件和脚本。早期做法是把所有工具说明、调用约定甚至示例都塞进系统提示或初始化上下文。工具数量一旦超过十几个，上下文先被固定消耗掉一大块，不同插件之间还容易出现命名冲突、调用方式互相干扰。

更关键的是，大部分任务根本不会用到所有能力。每次都全量注入，既拖慢响应，也增加模型误判概率。OpenClaw 的 Skills 机制想解决的就是这个问题：默认只暴露技能清单，等任务匹配到相关技能后，再展开该技能的正文、脚本和资源。

## 问题

简单把几个 `SKILL.md` 丢进目录，并不会自动实现按需加载。工程里常见三类问题：

- 触发描述太笼统，该命中时不命中，不该命中时乱加载。
- 技能正文写成大而全文档，加载后反而干扰决策。
- 脚本路径、权限没管理好，技能加载成功但执行失败。

所以 Skills 不能只当作“提示词文件”，要当作可测试的工程单元来管理。

## 做法与步骤

### 1. 固定目录结构

每个技能一个目录，`SKILL.md` 只放流程和指令，脚本放 `scripts/`，模板或静态资源放 `templates/`。例如：

```text
skills/
  report-export/
    SKILL.md
    scripts/
      export_report.py
    templates/
      report.md.j2
```

不要让一个技能目录里混入无关脚本，也不要跨技能引用资源。

### 2. 写好 frontmatter

`SKILL.md` 顶部用 frontmatter 描述元信息，其中 `description` 是触发匹配的主要依据。建议统一句式：当用户需要 X 或出现 Y 时，使用此技能完成 Z。

示例：

```yaml
---
name: report-export
description: Export a filtered report from SQLite into Markdown. Use when the user asks to export, generate, or download a report.
triggers:
  - export report
  - generate report
  - download report
tools:
  - sqlite
  - run_script
version: 0.1.0
---
```

`triggers` 可以补充常见说法，但不要堆太多，否则会和别的技能抢触发。`name` 用稳定的短横线命名，升级时尽量保持不变。

### 3. 渐进式披露

注册技能时，只把 `name` 和 `description` 注入清单。运行时命中技能后才读取 `SKILL.md` 全文。如果正文里需要调用脚本，统一使用相对路径，不写绝对路径。脚本内部也以技能目录为基准定位资源，避免路径漂移。

### 4. 配置挂载与权限

在 OpenClaw 配置中指定 skills 根目录，并限制可执行脚本类型。建议给技能脚本单独的执行环境，不要直接复用 Agent 的全部权限。需要写文件时，限制到白名单目录或临时目录。

### 5. 冒烟测试

为每个技能准备至少两个用例：一个是明确应该命中的表述，另一个是相近但不应命中的表述。验证清单匹配、正文加载、脚本执行三部分都正常。

## 踩坑点

- **描述太宽泛**：像“帮助用户处理数据”这种描述，会在无关场景也被加载。必须写具体任务和结果。
- **工作目录不一致**：运行时先确定当前目录，再通过相对路径找资源，不要在 `SKILL.md` 和脚本里各写一套路径逻辑。
- **换行符与 shebang**：Windows/Unix 换行和 shebang 差异，会导致脚本在本地能跑、换机器失败。建议统一 LF 并显式指定解释器。
- **多个技能描述接近**：触发路由会不稳定。可以把高频词从 `description` 中收敛，改由明确的 `trigger` 表达。
- **权限开太大**：技能脚本如果需要联网或写文件，不要直接给全局权限，使用最小授权。

## 可复用建议

- 一个技能只解决一条完整流程，不做多功能合集。
- 每次修改 `SKILL.md` 后，同步更新 `description` 和 `triggers`，并跑一遍冒烟测试。
- 记录技能命中日志：哪个技能被加载、耗时、是否执行脚本。未命中时可以通过日志反推失败在清单阶段还是正文阶段。
- 技能包纳入版本管理，升级时保持向后兼容的 `name` 和触发词。
- 先从一个高频重复任务开始做，跑通后再逐步迁移其他历史提示词。

## 总结

OpenClaw Skills 的价值不在于增加更多工具，而是用“清单 + 按需展开”控制上下文开销，让 Agent 在面对众多能力时先路由再加载。落地时关键动作是把技能当可测试的工程单元：结构固定、描述精确、权限最小、有冒烟验证。这样既保留扩展性，又不会让上下文被用不到的能力拖垮。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/b704697700713df6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/b55d44145ff5047c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/55d3c2f4a2fc69b2.png)

