---
title: Agent 的 tools.md：把本地环境差异变成可读契约
feedId: 33593
source: 综合讨论
publishedAt: 2026-08-17
---

## 背景

Agent 和普通脚本最大的区别在于：它需要根据任务动态决定调用哪些本地工具。比如一个自动化任务可能先要用 Python 解析数据，再调用 Playwright 截图，最后通过 MCP 服务写入笔记。但现实是，每台机器的本地环境都不同：

- macOS 和 Linux 的路径不一样，`python3` 可能来自 Homebrew、pyenv、系统自带或某个 venv；
- Node 版本、Chrome/Chromium 缓存目录、代理端口、MCP 服务端点都可能漂移；
- 同一个工具在不同版本下参数行为不一致，例如 Playwright 的浏览器路径在 CI 和本地就完全不同。

如果把工具说明硬编码进 system prompt，prompt 会迅速膨胀，换一台机器就失效。把路径写死在代码或脚本里，维护成本更高。工程上需要一个单一事实来源，用来描述“这台机器上有什么工具、怎么调用、有什么限制”。这就是 `tools.md` 的定位。

## 问题

常见的做法是把工具清单直接塞进 Agent 的系统提示词，例如：

```text
你是一个自动化助手。你可以使用 python3，路径是 /usr/bin/python3；可以使用 node，路径是 /usr/local/bin/node……
```

这样做有几个明显问题：

1. **不可移植**：换到 Windows 或另一台 Mac，路径全错。
2. **难以更新**：环境一变，就得改 prompt，测试和版本管理都很麻烦。
3. **信息过载**：所有工具说明都堆在 prompt 里，干扰模型推理。
4. **没有验证机制**：Agent 不知道工具是否真实可用，经常在错误路径上反复重试。

## 做法/步骤

### 1. 创建 `tools.md` 作为环境契约

在项目根目录或用户级配置目录放一个 `tools.md`，内容只描述事实，不写业务逻辑。结构建议按工具分块：

```markdown
# tools.md

## python
- path: $HOME/project/.venv/bin/python
- version: 3.12.4
- env: VIRTUAL_ENV=$HOME/project/.venv
- note: 优先使用 venv，系统 python3 仅作 fallback

## playwright
- browsers: chromium 位于 $HOME/Library/Caches/ms-playwright
- proxy: http://127.0.0.1:7890
- note: 无头模式需显式指定 --no-sandbox

## mcp
- filesystem: http://127.0.0.1:8765/mcp
- memory: http://127.0.0.1:8766/mcp
```

### 2. 让 Agent 在规划阶段先读 `tools.md`

在 system prompt 中增加一条明确指令，而不是把工具细节直接写进去：

```text
在规划任何需要本地工具的任务前，先读取项目根目录下的 tools.md。
如果所需工具不在 tools.md 中，先用 `command -v <tool>` 验证是否存在；
如果不存在，停止执行并报告缺失，不要自行猜测路径。
```

这样 Agent 的行为就变成“先查契约，再动手”。

### 3. 自动生成初稿，而不是手写

手写 `tools.md` 容易过时。建议用一个脚本扫描当前环境，输出 Markdown 初稿：

```bash
#!/usr/bin/env bash
echo "# tools.md"
echo
echo "## python"
echo "- path: $(command -v python3)"
echo "- version: $(python3 --version 2>&1)"
echo
echo "## node"
echo "- path: $(command -v node)"
echo "- version: $(node --version 2>&1)"
echo
echo "## playwright"
echo "- browsers: $(ls ~/Library/Caches/ms-playwright 2>/dev/null | tr '\n' ' ')"
```

脚本可以按需扩展，扫描 `mcp` 端点、浏览器路径、常用 CLI 工具。生成后人工补充 `note` 字段。

### 4. 建立更新与校验机制

环境变化时，重新运行生成脚本，并用 git diff 检查变更。也可以在 CI 中加一步校验：

```bash
grep -q "$HOME/project/.venv/bin/python" tools.md || echo "python path drift detected"
```

更严格的做法是让 CI 实际执行 `python3 --version`，与 `tools.md` 中记录的版本比对。

## 踩坑点

- **把密钥写进 `tools.md`**：很多人顺手把 API key、代理密码也写进去。一旦提交到仓库，就是安全事故。`tools.md` 应该只引用环境变量名，例如 `api_key: $OPENAI_API_KEY`，真实值放 `.env` 或系统环境变量。

- **路径转义和空格**：Windows 路径或带空格的路径容易破坏 Markdown 结构。建议统一用反引号包裹路径，或者干脆用 JSON/YAML 格式。如果团队跨平台，JSON 更稳。

- **文档过期**：`tools.md` 写完后没人更新，Agent 照着一个已经不存在的路径反复失败。解决方法是让 `tools.md` 尽量由脚本生成，并在 CI 中做一致性检查。

- **Agent 忽略 `tools.md`**：有些模型会“自作主张”猜路径。需要在 prompt 中反复强调，并且在工具调用层做硬校验：如果路径不在 `tools.md` 中，直接拒绝执行或先验证。

- **多平台差异**：不要写死 `/Users/alice/`，用 `$HOME` 或 `~`。MCP 端点如果绑定了 localhost，在容器或远程环境可能不可达，需要记录主机名或网络模式。

- **信息过度膨胀**：不要把系统里所有工具都 dump 进去。只记录和当前项目任务相关的工具，否则 Agent 会被无关信息干扰。

## 可复用建议

1. **分层覆盖**：全局 `~/.config/openclaw/tools.md` 放通用工具，项目级 `tools.md` 放项目专属工具。Agent 先读全局，再读项目级，后者覆盖前者。

2. **模板化**：在团队仓库中维护一个 `tools.template.md`，新成员 clone 后运行 `scripts/gen_tools_md.sh` 即可生成本地版本。

3. **与 `.env` 严格分离**：`tools.md` 只描述工具位置和调用方式，不包含任何 secret。需要密钥时引用环境变量名。

4. **版本锁定**：对影响行为的工具记录 major version，例如 Python 3.12 和 3.11 在某些库上行为不同。只写“3.x”是不够的。

5. **给 Agent 明确的失败路径**：在 prompt 中写清楚：如果工具缺失、版本不符或路径无效，不要尝试“修复”，先报告。这能避免 Agent 在错误方向上浪费大量 token。

## 总结

`tools.md` 本质上是一份本地环境的接口文档，它让 Agent 从“猜环境”变成“查契约”。核心价值不在于文档本身，而在于建立了一套可生成、可验证、可分层覆盖的机制。务实做法是：自动生成初稿、人工补充限制说明、CI 校验一致性、prompt 强制读取。做到这几步，本地环境差异就不会再成为 Agent 执行链路里的随机变量。

---

