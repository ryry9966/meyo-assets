---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 32525
source: 综合讨论
publishedAt: 2026-08-11
---

## 背景

在 OpenClaw Agent 的工程实践中，`tools.md` 是注册工具列表的核心文件——无论是 MCP 工具、本地脚本还是 HTTP 插件，都需要在此声明，以便 Agent 在运行时加载并暴露给 LLM 调用。但工具定义几乎总是依赖本地环境：API 密钥、文件路径、服务地址、可用性开关等。当项目从笔记本迁移到 CI 环境，再部署到生产服务器时，这些差异若被直接写死在 `tools.md` 中，会带来可移植性灾难、敏感信息泄露风险，以及维护上的持续痛苦。

一种常见的反模式是：将 `tools.md` 当作“最终配置”直接编辑，每个环境手动修改，甚至将密钥、内网地址一并提交到 Git。这种做法迟早会导致生产环境配置被覆盖、凭证泄露或 CI 工具加载失败。我们需要的是一个**声明式定义 + 环境注入 + 自动生成**的工程化流水线，让 Agent 的工具配置像应用代码一样可以被可靠地部署。

## 问题拆解

1. **硬编码凭证与路径**：`api-key=sk-xxxx`、`db-url=10.0.0.5:5432` 直接出现在 `tools.md` 中，切换机器就要手动改，且极易被提交到历史记录。
2. **工具开关分散**：部分工具仅在开发环境启用（例如 `local_file_search`），生产环境需要关闭，但条件判断散落在启动脚本或注释中，缺乏统一管理。
3. **格式脆性**：`tools.md` 是 Agent 依赖的配置源，一旦因为手动修改引入格式错误（如缩进、换行、未转义字符），会导致 Agent 启动失败且难以排错。
4. **环境变量传递链路断裂**：环境变量在 shell 中已设置，但 Agent 进程由 systemd、docker 或 supervisor 托管时，可能无法继承，造成工具调用时 `ENV_VAR` 为空。

## 做法：模板化生成 `tools.md`

推荐采用 **“定义源 + 模板引擎 + 环境变量注入”** 三层结构，将工具配置彻底与本地差异解耦。

### 1. 编写工具定义源（tools.def.yaml）
用结构化数据描述工具元信息，所有可变值使用 `${VAR}` 占位。
```yaml
tools:
  - name: doc_search
    description: Search internal docs
    command: ${TOOL_DOC_SEARCH_CMD:-/usr/local/bin/docsearch}
    args:
      - --index=${DOC_SEARCH_INDEX}
      - --token=${DOC_SEARCH_API_KEY}
    env:
      - DOC_SEARCH_AUTH=${DOC_SEARCH_AUTH}
    enabled: ${ENABLE_DOC_SEARCH:-true}
  - name: db_query
    description: Run read-only queries against metabase
    command: psql
    args:
      - ${METABASE_CONNECTION_STRING}
    enabled: ${ENABLE_DB_QUERY:-false}
```
注意：`${VAR:-default}` 语法允许指定默认值，但最终渲染脚本需支持这种展开，或统一在环境变量层面提供默认值。

### 2. 生成脚本（generate-tools-md）
使用一个简单的脚本（Python/Node/Shell）读取 YAML，替换占位符，输出符合 OpenClaw 规范的 `tools.md`。
```bash
# 通过 envsubst 或自定义模板渲染
export $(grep -v '^#' .env.${ENVIRONMENT} | xargs)
python3 render_tools.py tools.def.yaml > tools.md
```
渲染逻辑要处理布尔值 `enabled`：若为 `false`，该工具段不应写入 `tools.md`，从而动态控制工具列表。对于字符串值，需确保空格、换行被正确转义，避免破坏 Markdown 或后续解析。

### 3. 环境分层管理
创建 `.env.development`、`.env.staging`、`.env.production` 文件，分别存放对应环境的变量值。`.env` 文件本身加入 `.gitignore`，同时提供 `.env.example` 列出所有必需变量及说明，供新开发者复制使用。在 CI 和部署流程中，由平台注入对应环境文件（如 GitHub Actions 的 Secrets、K8s ConfigMap）。

### 4. 启动流程集成
将 `generate-tools-md` 作为 Agent 启动的前置步骤，例如在 `systemd` 单元中添加 `ExecStartPre`，或在 Docker 镜像的 `entrypoint.sh` 中运行。这样每次重启都会根据当前环境重新生成 `tools.md`，杜绝遗忘更新。
```
ExecStartPre=/usr/bin/python3 /opt/agent/scripts/render_tools.py /etc/agent/tools.def.yaml
ExecStart=/opt/agent/bin/openclaw-agent start --tools-file /etc/agent/tools.md
```

## 踩坑与避雷

- **环境变量存在但值为空**：某些 CI 环境会将未设置的变量定义为空字符串而非未定义。渲染脚本应区分“变量未定义”和“已定义为空”，对必须非空的变量做检查，退出并报错。
- **多行参数/特殊字符**：YAML 中的多行字符串可能与命令行参数预期不符。建议在定义源中避免多行拼接，必要时使用 Base64 编码传递，或确保渲染后在引号内正确转义。
- **`tools.md` 被误提交**：务必在 `.gitignore` 中明确忽略生成的文件，同时保留 `tools.md.example`（使用示例值）供参考。在 CI 的 lint 阶段增加检测，若仓库中存在生成的 `tools.md` 则构建失败。
- **systemd 环境变量**：systemd 默认不继承用户环境，需通过 `EnvironmentFile=` 指定，或使用 `Environment=` 显式声明。若变量中有百分号、引号等特殊字符，需按 systemd 规范转义。
- **工具路径跨平台**：`command` 尽量使用绝对路径或通过 `PATH` 中的命令名，避免依赖 `/usr/local/bin` 在不同发行版的差异。可在渲染时，根据当前平台的 `which` 结果替换，或由环境变量覆盖。

## 可复用建议

- **将渲染脚本设计为 CLI 子命令**：例如 `openclaw tool config generate`，既可在启动链中调用，也可手动重生成并验证。
- **增加脱敏输出**：提供一个 `--debug` 选项，在打印结果时自动隐藏 Token、Key 等敏感值（仅显示首尾字符），方便排查问题而不泄露信息。
- **分层覆盖**：对于有大量工具体系的大型项目，可引入 `tools.base.yaml` 和 `tools.dev.yaml`，用合并策略（类似 Kustomize）组合，而非在一个文件中用环境变量控制大量 `enabled` 开关。
- **在 README 中抽取出“环境就绪”步骤**：引导用户先复制 `.env.example`，填写变量，运行生成命令，再启动 Agent。将配置就绪流程标准化，减少“我运行了怎么报错”的 issue。

## 总结

将 `tools.md` 从硬编码的静态配置，转变为由定义源模板和环境变量动态生成的文件，实质上是引入了一层**配置编译期**。这层编译保证了环境差异被显式、可审计地管理，凭证不再残留于版本历史，工具列表也能随环境自适应。当 Agent 需要在多台机器、多个阶段间迁移时，你只需维护一份定义源和几份环境变量文件，而不必再对着 `git diff` 里的 `api-key` 字段提心吊胆。

实践中，核心仍在于那三个动作：**定义与值分离、生成自动化、生成结果不入库**。做到这三点，你的 Agent 工具配置就能像现代应用的配置管理一样可靠。

---

