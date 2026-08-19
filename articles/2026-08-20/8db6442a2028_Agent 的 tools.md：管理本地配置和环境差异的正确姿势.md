---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 33887
source: 综合讨论
publishedAt: 2026-08-20
---

## 背景：tools.md 为什么容易变成“环境差异放大器”

在 OpenClaw / Agent / MCP / 插件 的自动化实践里，`tools.md` 经常被当作入口配置：声明有哪些工具、MCP server 怎么启动、脚本入口在哪、本地路径和密钥怎么读。它看起来只是一个静态文件，但在多人协作、跨机器部署、从 macOS 换到 Linux 或 Windows 时，同一份 `tools.md` 往往会因为环境差异产生完全不同的行为。

常见现象是：A 机器上跑得好好的，B 机器一启动就报 MCP server 连接失败；或者 CI 里能跑，本地 GUI 启动的 Agent 却找不到环境变量。问题通常不在 Agent 本身，而在于 `tools.md` 把“本地环境依赖”和“工具定义”耦合在了一起。

## 问题：三类翻车现场

1. **硬编码路径**  
   `command: python /Users/alice/scripts/run.py` 或 `C:\Users\alice\...`。换一台机器或换一个用户就失效。

2. **环境变量未声明**  
   MCP server 需要 `ANTHROPIC_API_KEY` 或内部代理地址，但 `tools.md` 里没写清楚哪些变量是必需的。启动时 Agent 报错，用户不知道缺什么。

3. **工具版本漂移**  
   `command: python script.py` 依赖全局 Python 包，不同机器上版本不一致，导致同样的输入产生不同输出，甚至静默失败。

这些问题的共同点是：`tools.md` 被当成一次性脚本，而不是需要维护的接口定义。

## 做法：分层配置 + 自检脚本

### Step 1：把 tools.md 当接口，不写死实现细节

将 `tools.md` 拆成两层：静态工具清单和动态本地覆盖。静态部分只描述工具名、用途、启动命令的“接口”；动态部分由环境变量或本地文件填充。

```yaml
# tools.md 静态部分
tools:
  - name: local-notes
    description: Search local markdown notes
    command: ${NOTES_TOOL_CMD:-python ${OPENCLAW_HOME}/tools/notes.py}
    required_env:
      - OPENCLAW_HOME
    optional_env:
      - NOTES_VAULT_PATH
```

这里使用 `${VAR:-default}` 语法，给默认值，同时允许本地覆盖。`OPENCLAW_HOME` 是必需的，启动前检查。

### Step 2：用环境变量代替硬编码路径

绝对路径改为环境变量，例如：

- `OPENCLAW_HOME`：指向配置根目录。
- `PLUGIN_DIR`：插件目录。
- `MCP_CONFIG_DIR`：MCP server 配置文件目录。

提供 `.env.example`，不提交真实 `.env`。本地可以用 `tools.local.md` 或 `local.env` 覆盖，但不要提交到仓库。

### Step 3：为每个工具定义 required_env / optional_env / cmd

在 `tools.md` 里显式声明每个工具依赖的环境变量。Agent 启动前可以解析这些字段，检查缺什么，而不是等运行到一半才报错。

```yaml
- name: weather-mcp
  type: mcp
  command: npx
  args: ["-y", "@some/weather-mcp"]
  required_env:
    - WEATHER_API_KEY
  optional_env:
    - HTTP_PROXY
    - HTTPS_PROXY
```

### Step 4：写一个 healthcheck 脚本

在 `tools.md` 旁边放一个 `doctor.sh` 或 `check_tools.py`，用于启动前验证：

- 必需环境变量是否存在。
- 命令是否在 PATH 中。
- MCP server 能否成功启动并响应。
- 路径是否可访问。

Agent 可以在启动时执行该脚本，失败时给出明确提示。

### Step 5：MCP server 统一配置格式

MCP server 的启动通常依赖 stdio，但不同 shell 对参数解析不同。建议统一使用数组形式的 `args`，避免字符串拼接导致 Windows 下的引号问题。

```yaml
mcp_servers:
  - name: filesystem
    command: npx
    args:
      - "-y"
      - "@modelcontextprotocol/server-filesystem"
      - "${ALLOWED_DIRS}"
```

如果 `ALLOWED_DIRS` 包含空格，使用数组形式可以避免 shell 分词错误。

## 踩坑点

1. **GUI 启动的 Agent 加载不到环境变量**  
   macOS 的 launchd、Linux 的 systemd、Windows 的服务管理器都有独立的环境变量上下文。用户手动在 shell 里 `export` 的变量，GUI 启动的 Agent 看不到。解决办法：在 `tools.md` 中提供 `env_file` 字段，让 Agent 显式加载指定文件。

2. **相对路径基于 cwd 解析，而 Agent 的 cwd 不确定**  
   如果 `tools.md` 里写 `command: python ./scripts/run.py`，Agent 可能从项目根、用户主目录或临时目录启动。建议所有路径都基于 `OPENCLAW_HOME` 或绝对路径，不要使用 `./`。

3. **代理变量影响 MCP 连接**  
   `HTTP_PROXY` / `HTTPS_PROXY` 设置不当会导致 MCP server 连接超时。特别是 `NO_PROXY` 未包含本地地址时，本地 MCP server 可能被错误代理。建议在 `optional_env` 中明确代理变量，并在 healthcheck 中测试本地连接。

4. **Windows 路径空格和反斜杠**  
   Windows 路径包含空格时，字符串命令容易出错。使用数组 `args` 并在文档中标注平台差异。不要用反斜杠拼接路径，统一使用正斜杠或 `pathlib` 处理。

5. **工具返回非零退出码但 Agent 未处理**  
   有些工具失败时返回非零码，但 Agent 可能忽略，继续执行后续步骤。应在 `tools.md` 中约定工具失败时的行为：终止、重试、或降级。

## 可复用建议

- **分层覆盖**：`tools.md` + `tools.local.md` 或 `local.env`，本地敏感信息和路径不提交。
- **模板渲染**：用 `envsubst`、Jinja2 或 `opctl` 在启动前渲染生成实际配置。
- **提供 doctor 命令**：Agent 启动时先跑 `doctor`，输出缺依赖、缺变量、版本不匹配的诊断。
- **锁定工具版本**：MCP server 和命令行工具尽量固定版本，如 `npx -y package@1.2.3`，避免漂移。
- **跨平台测试**：在 macOS、Linux、Windows 各跑一次 healthcheck，把结果记录在 `tools.md` 的“平台差异”一节。

## 总结

`tools.md` 不是一个简单的工具列表，它是 Agent 与本地环境之间的接口定义。把路径、环境变量、平台差异、版本漂移这些问题提前显式化，可以让 Agent 在不同机器上表现一致，减少排查时间。核心思路是：**静态定义与本地覆盖分离，环境变量显式声明，启动前自检。** 这样，`tools.md` 才能真正成为可维护、可复用的工程配置，而不是每次换环境都要改一遍的一次性脚本。

---

