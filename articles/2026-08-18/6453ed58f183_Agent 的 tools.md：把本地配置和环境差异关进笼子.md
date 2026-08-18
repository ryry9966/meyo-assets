---
title: Agent 的 tools.md：把本地配置和环境差异关进笼子
feedId: 33710
source: 综合讨论
publishedAt: 2026-08-18
---

# Agent 的 tools.md：把本地配置和环境差异关进笼子

在 OpenClaw 以及各类 Agent + MCP + 本地插件的项目里，`tools.md` 经常被当作一份“工具清单说明”来写。写着写着就变成一段段自由文本，真正出问题时没人去看它。实际上，`tools.md` 如果能承担起**本地配置与环境差异的契约**角色，很多“我这跑不通”“换个机器就挂”的问题会少很多。

## 背景

一个典型 OpenClaw Agent 会依赖多种本地能力：ffmpeg 处理音视频、pandoc 转文档、sqlite3 查本地库、python 跑脚本，还可能通过 MCP 拉起本地服务。这些工具在不同开发者机器上路径不同、环境变量不同、版本不同。Agent 在调用工具时，如果只是简单 `subprocess.run("ffmpeg")`，那 PATH 没配好就崩；如果硬编码 `/usr/local/bin/ffmpeg`，换台 Linux 或同事的 Windows 就失效。

`tools.md` 正好可以成为这些工具的**单一事实来源**：声明工具的命令、环境变量、路径解析规则和验证方式。它不只是文档，而是配置约定的一部分。

## 问题

1. **工具配置散落**：一部分在代码里写死，一部分在 `.env`，一部分在系统环境变量，一部分在 README 的口口相传里。
2. **环境差异放大**：macOS 用 `brew` 装的路径、Linux 用 `apt` 装的路径、Windows 的 `.exe` 后缀和路径分隔符，都会导致同一份 Agent 配置出现不同行为。
3. **MCP 配置绕过**：MCP 服务本身有自己的配置文件，里面也可能写死命令路径。如果不和 `tools.md` 联动，两套配置互相打架。
4. **问题难定位**：报错信息往往是 `FileNotFoundError: ffmpeg`，但你不知道是该装、该配 PATH、还是该改 `tools.md`。

## 做法/步骤

### 1. 让 `tools.md` 成为工具契约

在项目根目录或 `.openclaw/tools.md` 中，为每个工具写一段结构化说明。示例：

```markdown
## ffmpeg
- 命令: ffmpeg
- 参数: -hide_banner -loglevel error -i {{INPUT}} {{OUTPUT}}
- 环境变量: OPENCLAW_FFMPEG_BIN (默认: ffmpeg)
- 路径解析: 优先使用 `OPENCLAW_FFMPEG_BIN`，其次使用系统 PATH
- 验证方式: `ffmpeg -version`
- 最低版本: 4.4
- 说明: 用于音视频转码，需要在服务端本地可用
```

这种结构不是为了好看，而是让**人和脚本都能读**。后面的校验脚本可以直接解析这些字段。

### 2. 统一环境变量前缀

给所有本地工具配置统一前缀，例如 `OPENCLAW_TOOL_*`。好处是：
- 一眼能看出哪些变量属于 Agent 的本地工具配置，不会和系统变量混淆。
- 方便在 Docker、CI、本地 shell 中统一注入和覆盖。

优先级建议固定为：**进程内注入 > 项目 `.env` > 用户 shell profile > `tools.md` 中的默认值**。把优先级写进 `tools.md` 的说明区，避免“我明明改了环境变量为什么没生效”的困惑。

### 3. 用占位符表达路径，而不是写死

在 `tools.md` 中，路径字段使用占位符：

```markdown
- 可执行文件: {{OPENCLAW_FFMPEG_BIN}} 或 {{PROJECT_ROOT}}/vendor/ffmpeg/ffmpeg
- 工作目录: {{PROJECT_ROOT}}/tmp/ffmpeg
```

代码或启动脚本在读取 `tools.md` 时，先替换占位符，再执行。这样换机器只需要改环境变量，不用改文档和代码。

### 4. 加一个验证脚本

可以写一个 `openclaw tools verify` 或简单的 shell 脚本，遍历 `tools.md` 中声明的工具：
- 检查环境变量是否存在，或回退到默认值；
- 尝试执行 `版本/验证命令`；
- 输出缺失、版本过低、路径无效的明确报错。

这样新成员克隆项目后，第一件事不是“跑 Agent 然后看哪里报错”，而是先跑一遍工具校验，问题提前暴露。

### 5. 把 `tools.md` 纳入版本控制

这是最关键的一步。`tools.md` 必须和代码一起提交，而不是留在某个人的本地笔记里。配置变更通过 PR 评审，所有人看到工具依赖的变化。

## 踩坑点

- **绝对路径写进 `tools.md`**：换机器必炸。要么用环境变量，要么用 `{{PROJECT_ROOT}}`。
- **环境变量覆盖顺序混乱**：有的地方读系统变量，有的地方读 `.env`，有的地方读 `tools.md` 默认值。最好在文档中明确优先级，并在代码里统一实现。
- **忽略 Windows 兼容性**：路径分隔符、`.exe` 后缀、命令是否存在差异。即使团队没人用 Windows，写 `tools.md` 时也尽量保持跨平台表述，避免以后接 CI 或新成员时踩坑。
- **MCP 配置和 `tools.md` 脱节**：MCP server 的命令路径写在 MCP 配置里，`tools.md` 又写了一遍，改了一处另一处没改。建议在 `tools.md` 中引用 MCP 配置的生成方式，或者让 MCP 配置也由 `tools.md` 派生。
- **文档和实现漂移**：嘴上说“先读 `tools.md`”，代码里还是硬编码。需要定期跑验证脚本，或者在 CI 中加入对比步骤。

## 可复用建议

1. **准备模板仓库**：把 `tools.md` 的结构化模板、验证脚本、占位符替换逻辑做成一个最小模板，新项目直接复用。
2. **CI 跨平台验证**：在 GitHub Actions 或自建 CI 上跑 `openclaw tools verify`，分别测试 macOS、Linux、Windows（如果条件允许）。这比事后排查高效得多。
3. **写最小可复现示例**：每个工具在 `tools.md` 里附一条最小调用示例，例如 `ffmpeg -i input.mp4 output.wav`。一来方便新人验证环境，二来出问题时能快速定位是参数问题还是环境问题。
4. **分层配置**：工具默认值放 `tools.md`，环境差异放 `.env` 或环境变量，机器特定配置放用户目录下的 `.openclaw/tools.local.md`。层级清晰，避免互相覆盖。

## 总结

`tools.md` 不是“工具使用说明书”，而是 Agent 本地环境的**契约**。它告诉 Agent 每个工具怎么找、怎么调、怎么验证，也告诉开发者环境差异应该往哪里填。用结构化写法、统一环境变量前缀、占位符路径和验证脚本，可以把“我这跑不通”变成一个能在几分钟内定位和修复的环境问题。

在 OpenClaw 这类本地工具链复杂的 Agent 项目里，越早把 `tools.md` 当成基础设施的一部分，后期维护成本就越低。

---

