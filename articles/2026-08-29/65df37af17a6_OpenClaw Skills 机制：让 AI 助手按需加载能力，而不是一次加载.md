---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力，而不是一次加载全部插件
feedId: 35256
source: 综合讨论
publishedAt: 2026-08-29
---

# OpenClaw Skills 机制：让 AI 助手按需加载能力，而不是一次加载全部插件

## 背景

在 OpenClaw 里接入 MCP、自定义插件或脚本后，很容易陷入“能力堆叠”。配置里挂十几个工具、API、文件处理器，每次会话都全量注册。结果上下文提示词变长、token 翻倍、冷启动变慢，甚至出现工具名冲突或权限误用。

Agent 不是不够强，而是加载了太多当前任务不需要的东西。

## 问题

静态加载和动态能力之间存在矛盾。完全动态发现会增加首次响应延迟；完全静态加载则污染上下文、浪费资源。OpenClaw 的 Skills 机制想解决的是：能力定义保留在磁盘或注册表中，但只有用户意图或任务触发时才挂载进当前会话，执行完可释放。

本质是把“能力发现”和“能力执行”分离。发现阶段只保留轻量元数据，执行阶段才加载真正的入口。

## 做法/步骤

### 1. 目录结构与 manifest

在 `skills/` 下每个能力一个目录，包含 `SKILL.md` 或 `skill.yaml`。manifest 至少声明 `name`、`description`、`entrypoint`、`triggers`、`permissions`、`dependencies`。

`description` 要写清楚“这个技能解决什么问题、何时不要用”，而不是泛泛的“帮助用户”。触发边界靠它来区分。

```yaml
name: pdf-summarizer
description: 当用户需要总结 PDF 内容时使用。不要用于扫描、OCR 或编辑 PDF。
triggers:
  - "总结 pdf"
  - "提取 pdf 摘要"
entrypoint: ./run.py
permissions:
  - read:file
dependencies:
  - pypdf
```

### 2. 注册与匹配

主进程启动时扫描 `skills/` 目录，只把 `name + description + triggers` 加载到 registry，`entrypoint` 不加载。用户消息进来后，先做一次轻量匹配：可以使用关键词、正则或小模型意图分类。匹配到 skill 后，再懒加载 `entrypoint`，注入到当前会话工具列表。加载完打印日志：

```
loaded skill=pdf-summarizer cost=12ms
```

这样能观测加载耗时，方便后续优化。

### 3. 触发边界

`triggers` 不宜太多；每个 skill 3-8 个触发短语即可。同时要配置负向规则，例如“不要与普通文本总结混淆”。否则用户说“帮我总结这段文字”可能误触 PDF 技能。

### 4. 卸载与回收

任务完成后，如果 skill 没有显式需要常驻，应允许卸载，释放内存和上下文。会话级挂载比全局挂载更合适：默认会话结束后自动卸载；长会话可设置 TTL，超过 N 轮未使用则回收。

## 踩坑点

- **描述太相似导致误加载**：比如两个技能都写“处理文件”，匹配时容易同时触发。解决办法：描述里加“什么情况下不要用”。
- **依赖未提前安装**：懒加载时才 `import` 库，可能因为缺依赖直接 500。建议在 manifest 声明依赖，并在注册阶段做 dry-run 或健康检查。
- **热加载缓存问题**：直接改 skill 文件，主进程可能因为缓存仍加载旧版本。建议以 skill 目录 hash 作为版本号，内容变化后重新加载。
- **权限边界不清**：自动加载技能如果带写权限，可能误操作文件。应按最小权限原则，且加载时让用户确认高权限动作。
- **状态污染**：一个 skill 在会话里设置了全局变量或改了工作目录，卸载时没恢复，影响后续技能。卸载阶段要有 cleanup 钩子。

## 可复用建议

1. 先做“手动优先 + 自动建议”两步：用户可用 `/skills` 查看可用技能，手动加载；同时系统根据意图建议加载，但不自动执行高权限技能。
2. 记录每次加载命中、耗时、误触发情况。根据日志收缩 `triggers`。
3. 限制单会话同时加载技能数，比如 3-5 个，超出时用 LRU 策略卸载不常用的。
4. 把 skill 的 `description` 当作 prompt 的一部分参与匹配，而不是只靠关键词；会让误触发率下降。
5. 为每个 skill 写一个 smoke test：模拟一条典型输入，看是否能正确加载、执行、卸载。

## 总结

按需加载不是简单地把 `import` 延迟，而是把“能力发现”和“能力执行”分离。OpenClaw Skills 的真正收益，是让 Agent 上下文保持小而准，同时不牺牲工具箱的丰富度。实现起来最难的往往不是加载机制，而是描述、边界和卸载清理。先把 registry 和日志做好，再谈自动匹配优化。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/63a7dfa20bb3ee6b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/8aa945773f8d6a3e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/a91acfd9fe612b6c.png)

