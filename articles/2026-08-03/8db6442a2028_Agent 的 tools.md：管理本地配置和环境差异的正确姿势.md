---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 31412
source: 综合讨论
publishedAt: 2026-08-03
---

## 背景

在 OpenClaw 生态中，`tools.md` 是 Agent 的工具清单——它定义了可用工具的名称、描述、参数 schema，以及最重要的：**工具如何连接到真实世界**。这最后一个部分往往意味着要写上文件路径、API 端点、数据库连接串、第三方 key 等。而一旦工程从单人本地开发扩展到多人协作、多环境部署，同一个 `tools.md` 里“写死”的配置就会变成绊脚石。

最常见的痛点是：开发环境指向本地 SQLite 文件、用 mock 的天气 API key；测试环境连 staging 服务；生产环境则需要真正的凭证和集群地址。如果每次切换都要手动改 `tools.md` 并小心翼翼地避开版本控制，不仅低效，还极容易把 secret 带上生产或误提交到 Git。

因此，我们需要一套务实、可复现、安全的方式，让 `tools.md` 能适应环境差异，同时保持工具定义的可读性和可维护性。

## 问题拆解

本质上，`tools.md` 是一个“静态描述文件 + 动态配置值”的混合体。静态部分（如工具名、描述、类型约束）可以提交到仓库；动态部分（路径、密钥、环境特定参数）则必须与运行环境绑定。

若不做分离，会遇到四类典型故障：

1. **硬编码泄密**：API key 直接写在文档里，被误推远程；
2. **环境漂移**：在开发机调通的工具，换一台机器或进入 CI 就找不到文件；
3. **维护混乱**：多人同时修改同一个配置文件，合并冲突频发；
4. **启动脆弱**：缺少变量时 Agent 直接抛出异常，而没有给出清晰的缺失项提示。

## 工程化做法

### 1. 使用变量占位符代替硬编码值

在 `tools.md` 中，凡是环境有关的值，一律使用 `${VAR_NAME}` 或 `{{VAR_NAME}}` 占位。例如：

```markdown
## tool: local_file_reader
- path: ${DATA_DIR}/input.txt
- encoding: utf-8

## tool: weather_api
- endpoint: ${WEATHER_API_ENDPOINT}
- api_key: ${WEATHER_API_KEY}
```

OpenClaw 运行时会解析这些变量并替换为实际值。你需要确认框架支持的占位符语法，通常遵循 POSIX 变量展开风格。

### 2. 提供环境变量文件模板

为每个运行环境准备一份 `.env` 文件，**永远不提交含有真实秘密的 `.env`**。将模板命名为 `.env.example` 提交到仓库，其中标注哪些是必填、哪些可选：

```bash
# .env.example
DATA_DIR=./data
WEATHER_API_ENDPOINT=https://api.weather.com/v1
WEATHER_API_KEY=<必填：在天气服务后台申请>
```

本地开发时可复制为 `.env` 并填入实际值。

### 3. 运行时注入：direnv / dotenv / 系统环境

- **本地开发**：推荐使用 `direnv`，进入项目目录自动加载 `.env`，离开时自动卸载，避免污染全局环境。
- **容器化部署**：通过 Docker Compose `env_file` 或 Kubernetes ConfigMap/Secret 将变量注入容器。
- **CI/CD**：在 GitHub Actions/GitLab CI 的 Settings → Secrets 中设定变量，流水线中导出即可。

对于多行内容（如私钥），务必使用 base64 编码或直接挂载文件，避免在环境变量中直接放置未转义的换行符。

### 4. 启动时校验，失败即止，给出清晰提示

这是很多团队忽略的环节。建议编写一个轻量的前置校验脚本（可以是 Shell，也可以是 Python），在 Agent 启动前检查所有关键变量是否已设置且格式合法：

```bash
#!/bin/bash
REQUIRED_VARS=(DATA_DIR WEATHER_API_KEY DB_URL)
for var in "${REQUIRED_VARS[@]}"; do
  if [ -z "${!var}" ]; then
    echo "错误：环境变量 $var 未设置，请检查 .env"
    exit 1
  fi
done
```

这样可以将“运行时找不到变量”的诡异错误转化为明确的配置缺失提示，大幅降低排障成本。

### 5. 区分“配置结构”与“环境数据”

更进一步，可将 `tools.md` 拆分为基础定义和后端配置。基础定义（如工具元信息）放在 `tools.base.md` 中，不同环境的覆盖值放在 `tools.{env}.md` 中，由启动脚本根据 `APP_ENV` 合并生成最终的 `tools.md`。这种做法适合环境差异巨大且工具数量多的项目。

## 踩坑记录

- **路径分隔符跨平台**：Windows 用反斜杠，Linux 用正斜杠。在 `tools.md` 中使用正斜杠，并在运行侧由程序做 normalize；或使用 `pathlib` / `os.path.join` 动态拼接，不要硬编码路径格式。
- **变量未展开**：部分 YAML/JSON 解析器不会自动展开 `${}`。确认 OpenClaw 的变量展开层是在解析文本之前还是之后执行，必要时调整占位符语法。
- **注释掉的值**：用 `#` 注释说明时注意不要破坏多行变量，例如私钥变量如果被折行且中间插入了注释符号，会变成无效值。
- **CI 中 “$” 被 Shell 提前展开**：在 GitHub Actions 的 `env:` 字段中写 `${VAR}` 需要转义或采用其他写法，避免被 CI 系统视为工作流变量而非目标程序的环境变量。

## 可复用建议

1. **优先使用环境变量而非文件配置**：对 Agent 这种偏向程序化运行的工具链，环境变量比配置文件更容易注入，也更容易被编排系统管理。
2. **一套 `tools.md` 配合多套 `.env.xxx`**：通过 `ENV=production` 切换 env 文件名是一种简单可靠的模式。
3. **将敏感变量与普通配置分离**：敏感的 API key 放入 Secret 管理服务（如 Vault、AWS Secrets Manager），启动时拉取并注入环境；非敏感的路径、超时等参数直接写在 `.env` 中。
4. **文档化**：在项目根目录的 README 中明确列出所有需要配置的变量，并说明获取方式，让新成员能 5 分钟内跑起 Agent。

## 总结

`tools.md` 是 Agent 的手和脚，环境差异管理是让它能跑在任何地方的中枢神经。通过变量占位符 + 环境变量注入 + 启动校验的方式，我们实现了配置与代码分离、多环境无痛切换、安全凭证管理。这个方案足够轻量，不需要额外服务，适合大部分小型到中型规模的 OpenClaw 实践。记住一条铁律：**所有可能随环境变化的值，都不应该硬编码在 tools.md 里。**

---

