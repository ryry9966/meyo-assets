---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 33892
source: 综合讨论
publishedAt: 2026-08-20
---

## 背景

在 OpenClaw 里，AI 助手的能力主要来自系统提示词、MCP 工具、插件和自动化脚本。功能做多之后，很容易出现一种情况：把所有工具和说明都塞进常驻上下文。结果就是首包变慢、token 成本上升、工具选择混淆，甚至权限边界模糊。

OpenClaw Skills 的定位是把一组相关能力打包成可被按需加载的单元。它不是简单地把 prompt 拆开，而是把「何时用、怎么用、用什么工具/脚本、需要什么环境」一起描述清楚，由加载器在运行时决定是否启用。

## 问题

如果只是写一个很长的 system prompt，罗列所有技能，会有几个实际问题：

- 上下文膨胀：几十个技能说明常驻，挤占真正的任务空间。
- 工具冲突：不同技能可能暴露相似名称的工具，AI 容易选错。
- 维护困难：改一个小脚本，可能要重新测试整段全局提示词。
- 安全边界不清晰：不需要的技能也拥有部分执行权限，风险变大。

MCP 解决的是工具协议问题，但 MCP 工具本身不携带“什么时候该用、不该用”的完整运营知识。OpenClaw Skills 更像是在 MCP 之上再抽象一层，把提示词、脚本、工具引用和触发条件打包。

## 做法/步骤

以一个「PDF 表格抽取并转 CSV」的技能为例。

1. 建立技能目录

```
skills/
  pdf-table-extract/
    SKILL.md
    skill.yaml
    scripts/
      extract_tables.py
```

2. 写 skill.yaml 描述元信息与触发条件

```yaml
name: pdf-table-extract
version: 0.1.0
description: >
  Extract tables from PDF files and convert them to CSV.
  Use when the user asks to pull tabular data from a PDF.
  Do not use for scanned images without OCR.
tools:
  - mcp:filesystem
  - mcp:pdf-reader
scripts:
  - scripts/extract_tables.py
env:
  - PDFTOTEXT_PATH
```

这里的关键是 description 同时写清楚“什么时候用”和“什么时候不用”。加载器根据意图匹配决定是否注入。

3. 写 SKILL.md 作为运行时提示词/操作说明

```markdown
# PDF Table Extract

## Steps
1. Check input file type and select PDF reader tool.
2. Run extract_tables.py with `--pages` and `--output`.
3. Validate CSV row count and encoding.
4. If scanned PDF, stop and ask for OCR tool.

## Output
Return CSV path and first 3 rows preview.
```

保持 SKILL.md 简短，只写任务执行所需的信息，不写大段背景。

4. 在 OpenClaw 中注册并测试按需加载

通常在 agent 配置里声明 skills 路径，或通过 CLI 注册。测试时故意使用不同问法：

- “帮我把这个 PDF 里的表格提出来” → 应触发。
- “把这个扫描件转成文本” → 不应触发，因为描述里排除了 OCR。

## 踩坑点

1. 触发描述太宽  
如果把 description 写成“处理 PDF 相关任务”，很容易在用户只是问 PDF 页面数量时也加载。建议写“Uses: ... / Avoid: ...”，并测试正反例。

2. 脚本路径与工作目录  
技能被加载后，执行脚本时工作目录可能不是技能目录。脚本里不要使用相对路径定位文件，应该通过参数传入绝对路径，或在 skill.yaml 里声明 `working_dir`。

3. 环境变量未声明  
脚本依赖系统命令（如 pdftotext），但 skill.yaml 里没写 env，加载后运行失败。最好在加载阶段做一次依赖检查，失败时给出明确错误信息，而不是让 AI 乱猜。

4. 卸载不干净  
技能任务完成后，如果 loader 只是追加提示词而没有移除，后续对话可能仍受该技能指令影响。需要在技能生命周期里定义 `on_unload`，或让加载器支持会话级隔离。

5. 缓存与版本  
修改 SKILL.md 或脚本后，如果 OpenClaw 有技能缓存，可能会继续用旧版本。调试时先清缓存或强制重新加载。

## 可复用建议

- 一个技能只解决一个明确问题，不要做成“瑞士军刀”。
- description 里同时写 uses/avoids，能显著降低误触发。
- 工具和脚本尽量通过 MCP 或命令行标准输入输出交互，减少对全局状态的依赖。
- 技能包内放一个最小的 smoke test：加载后运行一个空跑命令，确认环境可用。
- 如果是团队使用，建议给技能加版本号和变更说明，方便回溯。

## 总结

OpenClaw Skills 机制的价值不在于“把 prompt 拆成文件”，而在于把能力封装成可匹配、可加载、可卸载的单元。它与 MCP 配合时，MCP 负责工具通信，Skills 负责“何时、为何、如何”使用这些工具。实际落地时，重点应放在描述质量和生命周期管理上，否则容易变成另一种形式的全局提示词堆积。

---

