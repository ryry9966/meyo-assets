---
title: OpenClaw Skills 机制：把 AI 助手的能力变成按需加载的模块
feedId: 34397
source: 综合讨论
publishedAt: 2026-08-23
---

## 背景

在 OpenClaw 里，Agent 的能力通常来自三部分：系统提示、工具/MCP、以及一堆脚本文档。随着项目推进，很容易出现“万能助手陷阱”——把所有命令说明、排障手册、自动化流程都塞进 system prompt 或工具列表。结果上下文被静态占满，模型开始漏看关键指令，工具选择也变差。

Skills 机制要解决的就是这个问题：不默认加载所有能力，而是在需要时按任务特征把某个能力包注入上下文。

## 问题

如果不做按需加载，常见症状：

- 每次会话都带上大量从未使用的技能说明，token 成本稳定上升；
- 工具命名空间拥挤，模型容易选错工具；
- 添加新能力时要改动全局 prompt，回归测试困难；
- 能力之间隐式冲突，例如两个脚本都叫 `deploy`。

## 做法/步骤

我目前在 OpenClaw 项目里采用这样的结构：

```
skills/
  git-rebase-recovery/
    SKILL.md
    scripts/
      detect_rebase.sh
    assets/
      example_log.txt
  k8s-pod-debug/
    SKILL.md
    scripts/
      pod_doctor.py
```

每个 skill 是一个目录，入口固定为 `SKILL.md`。文件头用 frontmatter 声明元数据：

```yaml
---
name: git-rebase-recovery
description: 处理 git rebase 冲突、中断和误操作恢复。触发词：rebase 冲突、git rebase 中断、rebase abort。不用于普通 merge 冲突。
triggers:
  include:
    - "rebase"
    - "冲突"
    - "git rebase"
  exclude:
    - "merge"
priority: 10
tools: [bash, git]
timeout: 60
---
```

加载逻辑拆成两层：

1. **索引层**：启动 OpenClaw 时只扫描每个 `SKILL.md` 的 frontmatter，把 name/description/triggers 注入调度提示。完整正文不进入上下文。
2. **执行层**：当用户输入或任务计划命中 include 且不命中 exclude 时，才把对应 `SKILL.md` 正文和资源路径加载进来。

触发匹配不建议用复杂模型推理，优先用关键词+任务类型规则。OpenClaw 侧可以保留一个 `skill_index.json` 缓存，避免每次扫描目录。

具体接入步骤：

1. 在配置里指定 `skills_root: ./skills`。
2. 写一个加载器：遍历目录，解析 frontmatter，校验 name 唯一、description 非空、triggers 不与其他 skill 过度重叠。
3. 把索引注入调度层，并在系统提示里保留“可用 skill 列表”，但不展开正文。
4. 运行时命中后，按需读取 `SKILL.md` 正文；如果 skill 声明了 `tools`，再激活对应工具权限。
5. 脚本调用统一走沙箱或当前工作区，避免 skill 内部写死绝对路径。

## 踩坑点

- **description 过宽**：比如写“帮助用户处理 git 问题”，会导致任何 git 相关对话都触发。description 必须写清适用和排除场景。
- **触发词太短**：`include: ["run"]` 会误触大量任务。至少用双词组合或加排除词。
- **只加载正文不加载资源**：有些脚本依赖同目录的 `assets/`，加载时要把 skill 根路径写入上下文，否则模型找不到文件。
- **命名冲突**：两个 skill 使用同样的 name 或工具别名，索引层要强制唯一，否则会出现不可预测的覆盖。
- **元数据与正文不一致**：frontmatter 说“用于 k8s”，正文却在讲日志清理，触发后反而误导模型。建议每个 skill 加一个 example 输入输出，加载前做快速校验。
- **过度拆分**：一个 skill 只做一件事是好的，但拆到一条命令一个 skill，会让索引层变大、调度变慢。按“完成任务的最小闭环”拆。
- **忽略超时与权限**：执行型 skill 一定要声明 timeout，脚本需要最小权限，否则排障 skill 可能变成故障源。

## 可复用建议

- 把 skill 当代码仓库管理：每个 skill 单独版本，`SKILL.md` 里写 `version` 和 `changelog`。
- 使用模板创建新 skill，模板包含 frontmatter 必填项、测试样例、目录结构。
- 和 MCP 工具分层：MCP 提供原子能力（查 pod、看日志、执行命令），Skill 负责编排流程和上下文。不要把所有逻辑都塞进 MCP 工具，也不要把 skill 做成又一个工具描述。
- 保留 dry-run 模式：输入一段用户消息，只输出命中了哪些 skill，不实际执行。调试触发规则非常有用。
- 监控加载日志：至少在 debug 级别记录“命中/未命中/加载耗时”，便于定位 token 异常。

## 总结

OpenClaw Skills 机制不是增加一种炫技插件，而是把“能力加载”变成可管理、可测试、可回滚的工程问题。核心是控制上下文边界：索引常驻、正文按需、脚本受控、触发明确。项目越小越应该早建立这套结构，避免后期 prompt 和工具列表失控。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/a30f8c4ed510fde7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/760a8eab3a5cf660.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/b4c6a08b29c4d68e.png)

