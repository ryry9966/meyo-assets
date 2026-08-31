---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 35568
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景

OpenClaw 作为 Agent 运行时，通常会接入 MCP 工具、内部 API、脚本、知识库等。随着能力增多，如果所有工具和提示词都常驻上下文，会出现几个典型问题：上下文窗口被占满、工具选择噪声增加、不同能力之间的指令互相干扰、维护成本直线上升。Skills 机制的目标是把一组相关能力封装成可独立维护的单元，只有匹配到用户意图时才注入对应提示词和工具。

## 问题

全量加载不是工程方案。比如一个同时接文件处理、浏览器自动化、数据分析的 Agent，如果每次对话都注入所有工具描述、参数 schema、示例和约束，token 消耗会非常可观，同时模型选错工具的概率也会上升。更麻烦的是，不同 skill 的指令可能互相冲突：一个要求“只输出 JSON”，另一个要求“自然语言回复”；一个要求“禁止删除文件”，另一个却需要批量重命名。另外，安全边界不清晰，所有 skill 脚本都暴露给模型，风险较高。

## 做法/步骤

### 1. 设计 skill 目录结构

每个 skill 一个目录，包含 `SKILL.md`（元数据与触发说明）、`scripts/`（可执行脚本）、`references/`（参考资料）、`assets/`（静态资源）。例如：

```
skills/
  file-ops/
    SKILL.md
    scripts/convert.py
    references/mime-types.md
  browser-automation/
    SKILL.md
    scripts/playwright_runner.js
```

### 2. 用 frontmatter 声明契约

`SKILL.md` 的 frontmatter 是 skill 的核心接口。示例：

```yaml
---
name: file-ops
description: 处理常见文件转换、重命名、压缩
when_to_use: 用户要求转换文件格式、批量重命名、压缩目录
allowed_tools: ["fs.read", "fs.write", "shell.run"]
inputs: ["source_path", "target_format"]
outputs: ["result_path"]
---
```

### 3. 在 OpenClaw 配置中注册

在 `openclaw.yaml` 中声明 skills 路径和触发策略：

```yaml
skills:
  path: ./skills
  default_mode: auto   # manual | auto | disabled
  max_loaded: 3
  cache_ttl: 300
```

### 4. 运行时按需加载

OpenClaw 先让模型判断用户意图，再从 skill 索引中匹配候选。匹配到后，把对应 `SKILL.md` 的指令和 `allowed_tools` 注入当前会话，执行结束后卸载或缓存一段时间。多个 skill 可以链式触发，但需要在 `SKILL.md` 中显式声明 `dependencies`。

### 5. 建立可观测性

记录每次触发了哪些 skill、触发理由、耗时、是否成功、错误信息。这些数据是后续调优触发条件的基础。

## 踩坑点

- **触发条件过宽或过窄**：过宽会导致 skill 频繁误加载，过窄则永远不触发。建议先用小样本测试，统计触发准确率。
- **SKILL.md 内容冗长**：把详细步骤放到 `references/` 中，`SKILL.md` 只写核心约束和接口，按需让模型再读取参考资料。
- **脚本路径硬编码**：使用相对 skill 目录的路径，由 OpenClaw 注入 `base_dir` 环境变量，避免跨平台问题。
- **权限未隔离**：skill 脚本默认不应有完整系统权限，最好通过白名单工具调用，或者在沙箱中执行。
- **状态残留**：某些 skill 会写临时文件或环境变量，执行完要清理，避免影响后续对话。
- **缓存问题**：skill 更新后，旧会话可能还缓存旧版本，需要版本号或失效机制。

## 可复用建议

- 把 skill 当成 API 设计：有清晰的输入、输出、错误码和边界条件。
- 先用 `describe` 阶段让模型输出意图，再用匹配器选择 skill，而不是直接让模型自由调用。
- 为每个 skill 写一个最小测试用例，验证手动触发能跑通，再开启自动触发。
- 准备一个 skill 脚手架模板，减少重复配置。
- 控制单次加载的 skill 数量，避免并发冲突和上下文膨胀。

## 总结

OpenClaw Skills 机制解决的是 Agent 能力扩展中的上下文和治理问题：不是让模型记住所有能力，而是在需要时精准加载。通过清晰的 skill 契约、按需注入和可观测性，可以显著降低长期维护成本。建议从一两个高频 skill 开始，验证触发准确率后再逐步扩大范围。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/c83445e31a980ae5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/06b622c48803a9a9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/2bd3dc49f888fc3b.png)

