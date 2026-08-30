---
title: OpenClaw Skills 机制：让 AI 助手从“全量装载”转向“按需装配”
feedId: 35455
source: 综合讨论
publishedAt: 2026-08-31
---

在 Agent、MCP 和插件实践里，最容易被忽略的问题不是“能力不够”，而是“能力常驻”。把一堆 MCP server、插件或工具全量接入后，启动变慢、上下文被无关工具说明占满、模型选错工具的概率上升，很多资源在 80% 的会话里根本没被用到。

OpenClaw 的 Skills 机制提供了一种不同思路：把能力拆成小目录，按需读取，不常驻上下文。Skill 本身是一段可以被定位、读取、执行的程序化说明和脚本，只有任务匹配时才会被注入。

## 一、背景：为什么需要按需加载

常见的组装方式有两种：

1. **全量注册工具/插件**：所有能力都暴露给模型，工具列表非常长。
2. **纯 prompt 预设**：把大量操作手册塞进 system prompt，模型容易丢失重点。

这两类方式都会带来上下文膨胀、工具选择退化、维护成本上升。Skills 的出发点是把“能力索引”和“能力内容”分开：系统只保留轻量索引，命中后再加载完整内容。

## 二、Skills 目录长什么样

一个 Skill 通常是一个独立目录，核心是 `SKILL.md`，目录结构大致如下：

```text
skills/
  pdf-extract/
    SKILL.md
    scripts/
      extract_tables.sh
    references/
      layout-notes.md
```

`SKILL.md` 里用 frontmatter 描述触发条件，正文写执行步骤：

```yaml
---
name: pdf-extract
description: Extract tables from PDF reports when user asks for tabular data, batch extraction, or report parsing.
when_to_use: user mentions PDF table, batch PDF report, or data extraction
not_for: scanned image OCR, PDF page editing
tools:
  - bash
  - file-read
entrypoint: scripts/extract_tables.sh
---
```

关键是 `when_to_use` 和 `not_for`。它们不是给用户看的备注，而是加载决策依据。描述写得越清楚，误加载越少。

## 三、加载与执行流程

实践中通常是这样一条链路：

1. **构建索引**：扫描 skills 目录，只读取 frontmatter 中的名称和描述，不加载正文。
2. **任务匹配**：用户请求到来时，将任务摘要与索引做匹配。
3. **注入正文**：命中后读取 `SKILL.md` 正文、必要 references 或脚本说明。
4. **执行工具**：如果涉及脚本，在受控工作目录下运行。
5. **降级回收**：任务结束后，Skill 正文不再继续占用上下文。

可以用命令做基本管理：

```bash
openclaw skills list
openclaw skills validate pdf-extract
```

验证命令最好纳入 CI 或本地提交钩子，避免 SKILL.md 格式错误导致后续加载失败。

## 四、踩坑点

### 1. 触发条件写得太泛

如果 `when_to_use` 只写“处理 PDF”，模型可能在任何 PDF 相关任务里都加载这个 Skill，哪怕只是重命名文件。应写明具体场景，并补上 `not_for` 反向约束。

### 2. 正文太长

不要把所有细节都塞进 `SKILL.md`。正文应控制在可快速阅读的范围，细节放 `references/`。模型在需要时可以二次读取，而不是一开始就被长文档淹没。

### 3. 脚本执行环境假设

脚本里不要默认存在全局命令、特定 Python 版本或绝对路径。优先用 shebang、显式声明依赖，并在工作目录内使用相对路径。首次运行前先人工 review 一遍脚本。

### 4. 权限边界不清

Skill 一旦被触发，就可能在本地执行命令。不要把 untrusted skill 直接放进自动发现目录。对于外来 Skill，至少看一眼脚本和 `tools` 声明，确认不会越界。

### 5. 多个 Skill 命名冲突

不同路径下出现同名 Skill 时，加载顺序可能不稳定。建议给 Skill 加命名空间或统一目录约定，例如 `extract/pdf-tables`。

## 五、可复用建议

- **索引与正文分离**：索引要小，正文要精，细节放 references。
- **做 progressive disclosure**：先给步骤，再按需读取参考文件，避免一次性注入。
- **给脚本加 dry-run**：让 Skill 支持 `--dry-run`，代理可以先验证命令形态，不直接产生副作用。
- **记录加载日志**：至少记录“为什么加载/为什么不加载”，方便定位误触发。
- **纳入版本管理**：Skill 目录、SKILL.md、脚本和 references 一起走 Git，避免环境漂移。

## 总结

OpenClaw Skills 机制本质上不是给模型增加更多工具，而是把能力拆成可定位、可验证、可回收的单元。它解决的核心问题是“上下文装得下”和“模型选得对”。相比全量暴露，Skills 更适合内部操作流程、批处理脚本、固定排查路径这类任务；外部资源接入仍然可以交给 MCP，不必用 Skill 强行代替。

按需加载不是银弹，但它至少让扩展能力时，不再以牺牲上下文和工具选择精度为代价。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/ac07a099d7631e14.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/e1d9de6b9267bdb4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/993bc6229ed22754.png)

