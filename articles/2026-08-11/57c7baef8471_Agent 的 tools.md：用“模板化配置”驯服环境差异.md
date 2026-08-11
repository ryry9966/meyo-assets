---
title: Agent 的 tools.md：用“模板化配置”驯服环境差异
feedId: 32505
source: 综合讨论
publishedAt: 2026-08-11
---

## 背景：tools.md 承受的最后一公里困局

在 OpenClaw / MCP / 插件化 Agent 的日常实践中，`tools.md` 往往承载了工具定义、参数 Schema、执行环境等关键信息。它可以是一份 YAML 声明，也可以是 Markdown 中的结构化代码块，用来告诉 Agent：你可以调用哪些能力，每个能力需要什么输入。

然而，一旦把 Agent 从个人笔记本搬上 CI/CD，或者在不同开发者之间流转，一个最容易被忽视却反复咬人的问题就会暴露：**本地环境差异**。

数据库连接串不同、API Base URL 不同、本地文件路径不同、可执行程序名不同（比如 `python` vs `python3`）。如果这些差异被硬编码在 `tools.md` 的静态内容里，轻则工具加载失败，重则 Agent 静默地用了生产密钥却连上开发库，调试起来如同在沼泽里找鞋。

## 问题拆解：三类环境差异

我把真实踩过的坑抽象成三种差异：

1. **路径型差异**  
   - 示例：`/Users/alice/project/data` vs `C:\Users\Bob\project\data`  
   - 后果：工具找不到输入文件，或者输出写到奇怪的地方。

2. **终端型差异**  
   - 示例：Shell 脚本里写死了 `/usr/local/bin/ffmpeg`，但实际在 Docker 里是 `/usr/bin/ffmpeg`，或者本地叫 `ffmpeg` 需要从 PATH 解析。  
   - 后果：Agent 报告命令执行失败，可排查下来命令本身毫无问题。

3. **凭证/端点型差异**  
   - 示例：`prod` 环境的 API Key、DB URI 与 `dev` 完全不同。  
   - 后果：不小心把生产凭证写到 `tools.md` 再提交到 Git，会引发生安全事件。

维护者通常的做法是手动在 `tools.md` 里改来改去，或用 `.env` 临时注入，但缺乏统一范式，工具定义变得模糊而不可靠。

## 做法：将 tools.md 变成“模板”，变量从环境注入

核心思路是：**静态描述 + 动态值分离**。让 `tools.md` 本身不包含任何敏感信息或绝对路径，只保留工具的结构，运行时通过环境变量渲染出最终配置。

### 步骤 1：定义占位符语法

选择一个与解析器兼容的占位符形式。为了避免 YAML 中 `${VAR}` 被误解析为空，我习惯使用双大括号 `{{VAR}}`（类似 Helm 模板），或者用明确的表达式 `$ENV:VAR`。

例如在 YAML 中：

```yaml
tools:
  - name: run_query
    command: sqlite3 {{DB_PATH}} "SELECT count(*) FROM events;"
    env:
      DB_PATH: "{{DB_PATH}}"
```

如果你用的是 Markdown 里的 JSON 块，`${VAR}` 也可以，但要确保 JSON 解析阶段不会对 `$` 做特殊处理。

### 步骤 2：建立变量清单文件

在项目根目录放置 `tools.vars.example` 或 `.env.tools.example`，列出所有变量并给出示例值：

```
# Data
DB_PATH=/home/user/data/events.db
DATA_DIR=/mnt/shared_data

# API
OPENCLAW_API_BASE=https://api.dev.openclaw.io
OPENCLAW_API_KEY=sk-dev-xxxxxxxx

# Binary overrides
PYTHON=python3
FFMPEG=/usr/bin/ffmpeg
```

这样做的好处：新成员或新环境部署时，有明确的 checklist，不会遗漏变量。同时这个清单不应该包含真实密钥，`.gitignore` 中要排除渲染后的配置。

### 步骤 3：运行时渲染

在 Agent 启动或工具加载阶段，加一个轻量级的模板渲染步骤。可以用一个十几行的脚本（Python/Shell）读取 `tools.md.template` 并替换占位符，输出 `tools.md` 供 Agent 使用。

例如用 `envsubst` 处理变量：

```bash
export $(cat .env.tools | xargs)
envsubst < tools.md.template > tools.md
openclaw-agent start --tools tools.md
```

更工程化的做法是把渲染集成到插件的 `on_load` 钩子里，但原理相同。

### 步骤 4：校验必填变量

在渲染脚本或 Agent 初始化中加入校验：如果某个关键变量为空，立刻中断并给出清晰的错误消息。例如：

```python
required_vars = ["DB_PATH", "OPENCLAW_API_KEY"]
for var in required_vars:
    if not os.getenv(var):
        raise SystemExit(f"Missing required env var: {var}")
```

这样能把配置错误提前暴露，而不是在工具调用阶段才出现诡异的 “command not found” 或连接超时。

### 步骤 5：用条件环境块处理差异

有些差异不是简单的一个值替换，而是工具本身的选择。比如 `dev` 环境需要额外的调试工具，`prod` 则不需要。可以在 `tools.md.template` 里用环境变量做条件加载：

```yaml
{% if ENV == 'dev' %}
- name: debug_shell
  command: echo "Dev mode active"
{% endif %}
```

但这种条件语法需要更强的模板引擎（如 Jinja2）。如果不想引入复杂依赖，可以直接维护两份模板：`tools.md.dev` 和 `tools.md.prod`，通过脚本选择渲染哪一份，注入对应环境变量。

## 踩坑点：这些细节最容易翻车

1. **路径分隔符陷阱**  
   占位符 `{{DATA_DIR}}` 本身可能是 Unix 风格的路径，在 Windows 上运行 Agent 时，如果工具内部又用字符串拼接了子目录，会生成混合斜杠的路径。解法：尽量在变量里提供完整路径，或者由工具内部的库函数负责路径拼接。

2. **敏感信息泄露到日志**  
   即使 API Key 不在 `tools.md` 里，但如果工具的命令行参数打印了完整命令，可能会把 Key 打出来。设置日志过滤器，对带有 `API_KEY`、`TOKEN` 字样的变量做脱敏。

3. **占位符与语法冲突**  
   在 Markdown 代码块中写 `{{VAR}}`，有些 Markdown 解析器会把它视为模板变量，导致渲染为空白。解决办法：使用代码块的语言标注为 `plaintext` 或 `bash`，并确保最终渲染工具定义前再做一次检查。

4. **Docker 环境下环境变量继承**  
   如果在 Dockerfile 里用 `ENV` 设置变量，启动容器时传入变量会覆盖，容易让人误以为没生效。建议在容器启动脚本中显式打印生效的变量名（不打印值），方便确认。

## 可复用建议：把“模板化配置”做成团队约定

- **在项目 README 中开辟一个章节**：“首次运行 Agent 的工具配置”，引导执行 `cp tools.vars.example .env.tools` 并填写。
- **提供 Makefile 目标**：`make tools-render` 自动渲染，降低新人上手成本。
- **采纳统一的变量命名空间**：比如所有变量加 `OPENCLAW_TOOLS_` 前缀，避免与系统变量冲突。
- **在 tools.md 模板中保留一条注释**，说明“此文件由模板生成，请勿直接修改”，减少手工修改源头文件的冲动。
- **CI 中做变量完整性检查**：脚本查 `tools.vars.example` 里定义的变量是否在 CI 环境变量中都设置了，防止缺失。

## 总结

`tools.md` 是 Agent 与本地环境交互的“接线板”。用模板化配置管理环境差异，本质上是把可变部分提炼为显式变量，让配置变得可读、可验证、可审计。它不会增加太多工作量，但能有效避免“在我机器上没问题”的尴尬，也能拦住不小心提交的生产密钥。工程化 Agent 的第一步，就是让工具定义不再依赖隐形的手动修改。

---

