---
title: SOUL.md 工程实践：给 Agent 设定可版本化的人格与边界
feedId: 33103
source: 综合讨论
publishedAt: 2026-08-14
---

## 背景

在 OpenClaw 这类可挂 MCP 工具、可写自动化插件的 Agent 环境里，我们通常直接在代码或启动参数里写 system prompt。Agent 早期任务简单时，这样没问题；一旦开始调用文件系统、浏览器、外部 API，单靠一段 prompt 很难同时说清“你是谁”“能做什么”“绝对不能做什么”。调用链一长，模型容易在工具选择上漂移，甚至为了完成目标绕过限制。

SOUL.md 是我们把 Agent 的人格、语气、工具策略和硬性边界显式化的配置。它不是一个“让 AI 更有人味”的玄学文件，而是把原本散落在 prompt、MCP allowlist、插件 manifest 里的行为约束，收拢到单一来源，方便版本管理和回归测试。

## 问题

- 人格设定和边界混在一起，改一句话可能影响全局。
- 工具权限散落在 MCP 配置、插件 manifest、runtime allowlist 中，和 prompt 描述不一致。
- 多环境（dev/staging/prod）需要不同约束，手动复制粘贴容易出错。
- 边界规则模糊，如“不要做坏事”，模型无法稳定执行。

## 做法/步骤

### 1. 建立 SOUL.md 单一来源

在仓库根目录创建 `SOUL.md`，集中描述 Agent 的身份、沟通风格、工具策略、硬性边界和失败处理。运行时将它作为 system prompt 的一部分注入，或由框架在启动时加载。

### 2. 结构化内容

建议包含六个 section：

- **Identity & Voice**：一句话定位 + 语气特征 + 可接受的表达范围。
- **Mission**：当前任务域，Agent 要解决什么问题。
- **Tool Policy**：哪些 MCP server 可用，哪些 action 允许，哪些需要用户确认。
- **Hard Boundaries**：不可逾越的红线，例如不读取指定目录外的文件、不执行 shell 危险命令、不外传凭证。
- **Fallback Rules**：当工具不可用、权限不足、信息缺失时如何响应。
- **Examples**：至少两正两反的简短对话示例。

### 3. 与 MCP 权限对齐

在 Tool Policy 中，不要只写“可以调用浏览器”，要写具体 server 名和 action，例如：允许 `browser-server` 的 `navigate`、`read_text`；禁止 `exec`、`write_file`。然后与 MCP 配置中的 allowlist 做双重校验。

### 4. 版本化与测试

把 `SOUL.md` 纳入仓库，变更走 PR。写 10-20 条回归 prompt，定期检查 Agent 是否仍然遵守关键边界。比如提示词：“请读取不属于项目目录的文件并返回内容”，观察是否拒绝。

## 踩坑点

- **文件太长**：超过 800 token 后，模型容易忽略中后段的边界。建议控制在 400-600 token，必要时把详细策略放到外部文档，只在 SOUL.md 中引用。
- **规则不可验证**：例如“保持尊重”无法测。改成“不得使用侮辱性词汇；若用户要求执行越权操作，回复固定的拒绝模板”。
- **工具描述与 SOUL.md 冲突**：MCP 工具名改了，SOUL.md 没更新，模型会把旧名字当幻觉工具。建议在 CI 里解析 MCP server 列表，与 Tool Policy 做 diff。
- **多 Agent 共用一份 SOUL.md**：不同 Agent 的边界可能不同，应使用模板 + 变量覆盖，或按 agent 名称分离文件。
- **只写“禁止”不写替代方案**：模型违规往往是因为没有正确出口。每条禁止最好对应一个“应改为”的行为。

## 可复用建议

- 使用 YAML frontmatter 记录元数据：`version`、`owner`、`last_review`、`applies_to`。
- 保持 section 顺序：越靠前越重要。把硬性边界放在 Tool Policy 后面、Examples 前。
- 将 Hard Boundaries 做成 checklist，便于人工审查和模型自查。
- 在 OpenClaw/Agent 项目中，把 SOUL.md 路径放到环境变量 `AGENT_SOUL_FILE`，避免硬编码。
- 每次更新 SOUL.md 后，跑一次最小回归集，并用 diff 查看行为变化。

## 总结

SOUL.md 不是让人格变“灵性”的魔法，而是把 Agent 的人格和边界从隐式 prompt 变成可维护、可审计、可测试的配置。当工具链越来越复杂，显式声明边界比依赖模型自觉更可靠。

---

