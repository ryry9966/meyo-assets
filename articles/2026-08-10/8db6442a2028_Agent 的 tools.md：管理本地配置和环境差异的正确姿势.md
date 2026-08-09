---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 32317
source: 综合讨论
publishedAt: 2026-08-10
---

## 背景

在 OpenClaw 生态里，Agent 的能力高度依赖外部工具。这些工具通常通过 `tools.md` 来声明可用的命令、参数、描述，以及 MCP 服务端的连接信息。一个典型片段可能是：

```markdown
## search_file
- command: /usr/local/bin/fd
- args: ["--hidden", "--type", "f", "$QUERY"]
```

看起来没问题。但当同一个仓库被不同的人、在不同机器上 clone，或者分别部署到开发机、CI 服务器和生产容器时，`/usr/local/bin/fd` 很可能并不存在 —— 开发机用 Homebrew 装在了 `/opt/homebrew/bin/fd`，CI 里的 fd 是通过 npm 全局安装的 `/home/runner/.npm-global/bin/fd`，生产容器则可能是 `/app/bin/fd`。

更麻烦的是，有些工具还需要 API Key、区域配置或内部域名。把它们直接写死在 `tools.md` 里，既难以迁移，也存在安全风险。这篇文章只讨论一个具体问题：**在 OpenClaw Agent 里，如何让 `tools.md` 适应不同环境，同时保持仓库干净、可复现。**

## 问题本质

Agent 的 `tools.md` 本质上是一个“契约文件”：它告诉 Agent 可以调用哪些外部能力。但契约里不应该写死实现细节，尤其是与运行环境强绑定的路径、端口和凭证。

常见的错误模式：

- **硬编码绝对路径**：`command: /home/zhangsan/bin/my-tool`
- **直接写入明文密钥**：`- env: ["OPENAI_API_KEY=sk-xxxx"]`
- **假设固定的端口或主机名**：`url: http://localhost:6666/api`
- **全量提交到 Git**，个人修改后产生大量冲突

这些事情都会让 Agent 变成一个“脆皮工程”——换一台机器就爬不起来，也很难被其他团队成员或社区用户复用。

## 正确姿势：环境变量 + 模板化

核心思路：**把随环境变化的部分抽离成变量，在 Agent 启动时注入，而不是固化在文档中。**

### 1. 在 `tools.md` 里使用占位符

OpenClaw 的 Agent 运行时会读取 `tools.md`，并允许使用 `$VAR` 或 `${VAR}` 形式的变量。因此我们可以把工具定义写成模板：

```markdown
## fd
- command: ${FD_CMD}
- args: ["--hidden", "--type", "f", "$QUERY"]
```

或者当工具本身不支持环境变量覆盖命令时，至少把可执行文件目录抽离：

```markdown
## fd
- command: ${TOOLS_BIN_DIR}/fd
```

然后，在 Agent 的启动配置（例如 `.env` 文件或 `agent.yaml` 的 `env` 段）中定义：

```bash
FD_CMD=/opt/homebrew/bin/fd
TOOLS_BIN_DIR=/opt/local/bin
```

这样同一份 `tools.md` 就可以工作在 macOS、Linux 以及各种容器环境中。

### 2. 分层管理配置

不要把所有变量都堆在一个全局 `.env` 里。推荐按用途拆分：

- `agent.env`：Agent 框架本身需要的变量（模型端点、日志等级）
- `tools.env`：工具路径、通用开关（如 HTTP_PROXY）
- `secrets.env`：各类 token（不纳入版本控制）

在 OpenClaw 的 `agent.yaml` 中可以通过 `env_file` 字段引用多个文件。同时，可以给不同环境准备不同的 `env_file`，例如：

- `envs/dev/tools.env`
- `envs/prod/tools.env`

这让环境差异显式化，审阅和修改都更安全。

### 3. 用脚本校验前置条件

即使使用了变量，Agent 启动时仍可能因为忘记设置变量而失败，而且报错信息通常不太友好（“command not found: ${FD_CMD}” 或直接崩溃）。一个低成本的做法是在启动前做一次快速检查：

```bash
#!/usr/bin/env bash
required_vars=("FD_CMD" "TOOLS_BIN_DIR" "MCP_SERVER_TOKEN")
for var in "${required_vars[@]}"; do
  if [ -z "${!var}" ]; then
    echo "Error: environment variable '$var' is not set."
    exit 1
  fi
done

# 进一步检查命令是否存在
if ! command -v "${FD_CMD}" &> /dev/null; then
  echo "Error: '${FD_CMD}' not found."
  exit 1
fi
```

把这个脚本挂在 Agent 的 `pre_start` 钩子里，能够把绝大多数环境问题拦截在启动阶段，而不是等到实际执行工具时才暴露。

## 踩坑记录

1. **变量替换不生效**  
   检查 `tools.md` 中是否使用了花括号或特殊字符，OpenClaw 的解析器对 `${VAR}` 和 `$VAR` 的支持可能不完全一致，尤其是路径中包含 `:` 或空格时。建议统一使用 `${VAR}`，并避免路径中的空格。

2. **docker 镜像中的路径差异**  
   如果想做一个既能本地跑又能容器化的 Agent，工具路径推荐使用类似 `/usr/local/bin/fd` 这种通用约定，而不是 `/home/user/.local/bin/fd`。为了让容器内外都能用，可以利用 `Makefile` 或 `docker-compose.yml` 中的 `environment` 来覆盖变量。

3. **Git 泄露敏感信息**  
   如果团队里有人习惯把 `.env` 文件提交上去，哪怕 `.gitignore` 写了也可能因为 `git add -f` 被意外包含。更保险的做法是将 `secrets.env` 模板化为 `secrets.env.example`，并提供默认假值。

4. **变量多了之后忘记文档化**  
   建议在 `tools.md` 顶部用注释形式列出所有需要的外部变量，例如：

   ```markdown
   <!-- 
     Required env vars:
       FD_CMD        - path to fd executable
       TOOLS_BIN_DIR - base dir for custom tool binaries
       MCP_SERVER_TOKEN - authentication token for mcp server
   -->
   ```

## 可复用建议

- **提供一份 `tools.md.template`**：把 `tools.md` 改成模板文件，并在 README 里说明如何根据本地环境生成最终文件。用户 clone 后执行一次 `make init` 或脚本即可。
- **使用 direnv 自动加载**：对于本地开发，在项目根目录添加 `.envrc`，自动加载 `tools.env` 和 `secrets.env`，避免每次手动 source。
- **CI/CD 中的 secrets 注入**：在 GitHub Actions 或 GitLab CI 中，将敏感 token 存储为受保护的变量，然后在 job 中写入 `secrets.env`，Agent 直接读取即可，无需修改任何代码。
- **锁定工具版本**：就算路径解决了，不同版本的工具行为也可能不同。建议在 `tools.md` 的注释里标注预期版本，并在校验脚本里加入版本检查（如 `fd --version`），进一步减少“环境差异幽灵”。

## 总结

`tools.md` 是 Agent 面向外部世界的接口约定。把它当代码一样对待，做好环境隔离，才能让 Agent 真正跨平台、可移植。核心原则只有一条：**不要相信任何一个环境的默认值，把差异交给变量，把校验交给自动化。** 做到这点，团队协作和后续的自动部署都会顺畅很多。

写完这篇，不妨回去检查一下你的 OpenClaw 项目里，是不是还有 `/Users/自己名字/...` 这样的路径。如果有，现在就是修正它的好时机。

---

