---
title: Agent 的 tools.md：让本地配置与环境差异不再隐身
feedId: 32769
source: 综合讨论
publishedAt: 2026-08-12
---

## 背景：工具在运行，谁在管它们的环境？

在 OpenClaw、MCP 客户端或自建 Agent 的工程实践中，我们会挂载许多本地工具：文件搜索、Shell 执行、数据库查询、内部 API 调用……这些工具几乎都依赖特定的本地环境——某个路径必须存在、某个环境变量必须设置、某个二进制文件必须在 `$PATH` 上。

一开始，我们在工具的 `run()` 函数里直接写死路径，或者被动地读 `os.getenv()`。当工具数量积累到 5 个以上，环境差异开始制造大量“摸不着头脑”的故障。某台机器上报 `FileNotFoundError`，另一台则是认证失败，而真正的根因往往只是环境变量名拼写错误或漏配。更糟糕的是，团队协作时，新成员拿到一份 Agent 代码，面对几十个工具，完全不知道要配哪些环境变量才能让所有功能跑起来。

这个问题本质上不是“工具写错了”，而是**工具的配置元信息被隐含在代码里，从未显式暴露**。我们需要一个可阅读、可解析、可审计的清单，把每个工具所需的运行环境说清楚。

## 做法：引入 `tools.md` 作为工具配置的单一事实来源

我们在 Agent 项目根目录下创建 `tools.md`，它不是简单的文档，而是一个**结构化的工具清单与配置要求声明**。Agent 启动时解析该文件，校验当前环境，只为满足条件的工具生成运行时配置。

一个典型的 `tools.md` 片段如下：

```markdown
# Tools Registry
## Tools
- name: local_file_search
  description: Recursively search local filesystem
  required_env:
    - SEARCH_ROOT
  defaults:
    SEARCH_ROOT: /home/agent/data
  optional_env:
    - SEARCH_IGNORE_PATTERNS
  notes: >
    SEARCH_ROOT must be an existing readable directory.
     Missing it will cause this tool to be disabled at startup.
- name: api_client
  description: Call internal REST API
  required_env:
    - API_KEY
    - API_ENDPOINT
  defaults:
    API_ENDPOINT: https://api.internal.example.com
  sensitive_env:
    - API_KEY
- name: sql_runner
  description: Execute read-only SQL queries
  required_env:
    - DB_CONNECTION_STRING
  availability_check:
    exec: "which sqlite3"
    expect_exit_code: 0
```

这个文件满足几个关键需求：

1. **声明依赖**：用 `required_env` 明确工具必需的配置项。
2. **提供默认值**：降低首次运行的配置成本，同时避免硬编码敏感信息（密码类只留占位符）。
3. **标记敏感字段**：`sensitive_env` 告诉日志系统这些值不能输出。
4. **环境能力检查**：`availability_check` 可用于验证二进制依赖是否存在。

Agent 在启动时读取这个文件，用一个轻量的解析层（几十行 Python/YAML 足够）生成工具清单，然后交叉比对当前环境变量。缺失必需项的工具会被自动禁用，并打印结构化日志：

```
[agent.init] Tool 'local_file_search' ready.
[agent.init] Tool 'api_client' DISABLED: missing env vars ['API_KEY'].
[agent.init] Tool 'sql_runner' DISABLED: command 'which sqlite3' failed.
```

最终只有通过校验的工具才会被注册给 Agent 的 planner 或 MCP 服务。这样，环境差异不再是隐性炸弹，而是启动时就能看到的清晰状态。

## 踩坑记录

在实际落地过程中，有几个地方容易翻车：

**格式漂移**：`tools.md` 是给人看的，但也是给程序解析的。多人协作时很容易出现缩进不一致、字段名拼错。我们最终为 `tools.md` 定义了一个简单的 JSON Schema，并在 CI 里用 `yamllint` + 自定义校验脚本做格式检查，避免“文档写了一套，代码解析另一套”。

**敏感信息泄露风险**：defaults 里绝对不能放真实密钥。可以在 `sensitive_env` 中标记，并在启动日志中打码。对于必须存在的密钥字段，default 只写 `"{{REQUIRED}}"`，Agent 检测到占位符时也会拒绝启动，防止误用。

**路径差异**：不同操作系统的路径表示不同。后来我们在 `tools.md` 中允许用 `$HOME`、`%APPDATA%` 等变量，并在解析层做统一展开。

**工具代码与声明不一致**：新增工具时，开发者可能只写了代码而忘记更新 `tools.md`。我们在工具注册装饰器里加了一个 `required_env` 参数，并在测试里对比代码声明与 `tools.md` 是否一致，不一致则 CI 报红。

## 可复用的工程建议

1. **用 `tools.md` 生成 `.env.example`**：写一个小脚本解析 `tools.md`，自动生成带有注释的环境变量模板文件。团队成员 clone 项目后只需 `cp .env.example .env` 并填充即可。
2. **封装为 MCP 资源**：如果你的 Agent 支持 MCP，可以将解析后的工具配置状态暴露为一个 Resource 或 Tool 本身，让 Agent 在对话中能自我诊断——“告诉我现在有哪些工具不可用，需要我做什么”。
3. **分层管理**：当工具数量继续增长，考虑按领域拆分为 `tools_db.md`、`tools_fs.md`，在总配置中引用，避免单文件膨胀。
4. **启动校验前置**：将环境校验逻辑放在 Agent 主循环之外，最好做成一个独立的 `check_tools.py`，可以在 Docker 构建、部署脚本中单独运行，早发现早修复。

## 总结

`tools.md` 这个模式不是在发明新技术，而是把工程中散落各处的隐性知识显式化。它让 Agent 系统的工具配置变得可读、可校验、可移交。对于有多工具、多人协作的自动化项目，这种“自文档化”的配置管理远比在代码里写死路径或被动依赖全局环境变量要可靠得多。

当你的下一个同事问“这个 Agent 在本地怎么跑起来”，你可以把 `tools.md` 扔给他，然后继续喝咖啡。

---

