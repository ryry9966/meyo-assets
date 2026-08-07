---
title: Agent 的本地配置说明书：用 tools.md 管理环境差异的正确姿势
feedId: 32044
source: 综合讨论
publishedAt: 2026-08-08
---

## 背景：被忽略的“最后一公里”

在 OpenClaw / MCP 的实践中，我们花大量精力打磨 tool 的接口设计、prompt 调优，但往往在部署到另一台机器或交接给同事时，工具直接罢工。常见症状：文件路径不存在、环境变量缺失、数据库连不上、Python 版本不对。排查一圈后发现，问题不在代码，而在本地环境差异。

Agent 调用 tool 时，对执行环境的上下文几乎一无所知。LLM 能推理，但它不知道你本机 `ffmpeg` 装在 `/opt/homebrew/bin` 还是 `C:\tools`。传统做法是把这类信息散落在 README、.env.example、甚至口头交代中，但 Agent 无法消费这些非结构化信息。于是，我们引入 `tools.md`——一份写给 Agent 看的本地配置说明书。

## 问题：配置熵与不可复现的失败

手工维护 tool 的本地依赖至少造成三个工程痛点：

1. **配置碎片化**：连接字符串记在 `.env`，路径写死在代码里，版本要求仅在安装脚本中体现。Agent 无从知晓这些约束，调用失败后只能返回模糊的错误。
2. **环境不可感知**：同样的 tool 在 macOS 与 Windows 上行为不同，但 Agent 完全不知道自己的运行平台。如果 tool 内部未做适配，调试就像开盲盒。
3. **敏感信息泄露风险**：某些实践会将含密钥的配置直接粘贴到 Agent 的 system prompt 中，增大了泄露面。

这些问题无法通过更好的模型或更复杂的 prompt 解决，需要一套轻量级配置信息路由机制，而 `tools.md` 正是为此而生。

## 做法：构建 Agent 可读的配置清单

我们可以在每个 tool 目录（或 MCP server 根目录）放置一个 `tools.md`，作为该工具集的“本地配置声明”。它面向两方读者：人类开发者与 Agent。文件由开发者维护，Agent 在需要调用工具前被指导去读取并遵守其中的环境约定。

### 1. 文件结构建议

一个生产可用的 `tools.md` 通常包含四个版块：

- **环境要求 (Environment Requirements)**：运行时依赖和版本。例如：
  ```
  - Python >= 3.10
  - ffmpeg 可执行文件应位于系统 PATH
  ```
- **配置项清单 (Configuration Reference)**：列出所有必要的环境变量及默认值，配合描述：
  ```
  | 变量名          | 必需 | 默认值            | 说明                       |
  |----------------|------|-------------------|----------------------------|
  | OPENAI_API_KEY | 是   | -                 | 用于 LLM 调用的 API Key     |
  | DB_PATH        | 否   | ./data/app.db     | SQLite 数据库文件路径       |
  ```
  这里只描述变量含义，不暴露真实密钥。
- **本地路径约定 (Local Path Conventions)**：注明不同操作系统下关键资源的典型位置，帮助 Agent 自动推理。如：
  ```
  - macOS: ffmpeg 通常位于 /opt/homebrew/bin
  - Windows: 可能需手动指定 FFMPEG_PATH 环境变量
  ```
- **常见故障排查 (Troubleshooting)**：列出过去常踩的坑，用条件判断形式写出，便于 Agent 做因果推断：
  ```
  - 若出现 "Connection refused"，检查 DATABASE_URL 是否指向运行中的 Postgres 实例
  - 若文件写入失败，确认 OUTPUT_DIR 是否存在且有写权限
  ```

### 2. 让 Agent 读取 tools.md

可通过两种方式将 `tools.md` 注入 Agent 的执行上下文：

- **System Prompt 注入**：在 Agent 的 system prompt 中明确指出 “调用工具前，优先阅读该工具所在目录的 tools.md，并严格遵循其中的环境设定”。部分框架支持动态注入工作目录下的文件内容。
- **作为 Tool 的元数据**：如果使用 MCP，可以在 tool 的 description 或者 `manifest.json` 中嵌入 `tools.md` 的摘要，或直接将其作为资源暴露给模型。

我们推荐第一种方式，并在 prompt 中提醒 Agent：*如果 tools.md 存在，不要假设默认环境，始终以它为准。*

### 3. 与 `.env` 的分工

`tools.md` 是*描述性*配置档案，`.env` 是*实际值*的来源。建议在 `tools.md` 中引用变量名，实际值从 `.env` 加载，这样 Agent 可了解配置结构，但不会接触敏感信息。同时，在 `.gitignore` 中排除 `.env`，但保留 `tools.md` 的版本控制，以便团队共享环境约定。

## 踩坑点

1. **位置定义模糊**：Agent 不知道从哪里开始查找 `tools.md`。最好在 system prompt 中明确指定“工具目录根”的定位逻辑，例如 `$MCP_WORKSPACE/tools/<tool_name>/tools.md`。
2. **Markdown 解析差异**：Agent 对表格、代码块的理解能力差异较大。实践中，用简单的键值对或列表比复杂表格更可靠。可以添加 `<!-- Agent: read this section for path config -->` 注释引导注意力。
3. **路径跨平台**：在 `tools.md` 中直接写 Unix 路径会导致 Windows 上调用失败。可以用环境变量完全替代硬编码路径，并在文档中说明各平台的推荐变量值。
4. **更新滞后**：开发者改了 `.env` 结构却忘记同步更新 `tools.md`。建议在 CI 中加入检查，比如对比 `tools.md` 中声明的变量与实际 `.env.example` 的差异。

## 可复用建议

- **模板化**：将以上四个版块做成项目模板，新建 tool 时直接复制修改。
- **敏感分层**：`tools.md` 永远不写密钥，只引用变量名；部署时由运维注入真实值。
- **与 MCP 集成**：如果使用 MCP，可将 `tools.md` 作为 `resources` 暴露，并让服务在启动时打印加载的环境摘要，方便 Agent 校验。
- **可观测性**：在 tool 的执行日志中记录“读取 tools.md 成功 / 失败”，当失败时让 Agent 进入降级模式，使用保守的环境探测。

## 总结

`tools.md` 是一种极低成本的本地配置治理实践，本质上是用人类可读、Agent 可解析的方式，显式化运行环境差异。它不替代 `.env` 或配置文件，而是为 Agent 提供了一层*可解释的元配置*，让工具调用从“猜测环境”走向“感知环境”。对于多平台协作、频繁交接的项目，这个过程节省的 debug 时间远超编写文档的成本。

---

