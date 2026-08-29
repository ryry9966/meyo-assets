---
title: Agent 的 tools.md：把本地差异关进配置，而不是散落在代码里
feedId: 35270
source: 综合讨论
publishedAt: 2026-08-30
---

## 一、背景：Agent 的“可运行”是被本地环境包裹的

在 OpenClaw、MCP 客户端或插件自动化里，很多失败不是模型推理错，而是工具压根没在本地环境里跑起来。同一个 Agent，在 A 机器上能调 ffmpeg，在 B 机器上连二进制都找不到；在 macOS 上能读 `~/data`，到 Linux 容器里路径全变；本地开发时端口 8080 没事，部署后和别的服务撞车。

`tools.md` 的价值，不是把所有命令堆进去，而是把“本地配置和环境差异”显式化。它应当是一份声明：这个工具需要什么二进制、什么环境变量、什么工作目录、什么平台分支。代码里不要散落路径和密钥，差异只在配置层解决。

## 二、问题：三类最容易翻车的差异

1. **路径差异**  
   `python3` 还是 `python`，`/opt/homebrew/bin/ffmpeg` 还是 `/usr/bin/ffmpeg`，Windows 反斜杠还是 Unix 正斜杠。

2. **环境变量与密钥**  
   开发机有 `.env.local`，服务器只有系统环境变量；API Key 不应该写进 `tools.md`。

3. **运行时上下文**  
   MCP stdio 子进程是否继承 Agent 的环境？相对 cwd 到底相对谁？端口、超时、工作目录在每台机器上可能完全不同。

## 三、做法：三层配置 + 环境变量解析

不要试图写一个“万能 tools.md”。建议拆成三层：

```markdown
# tools.md —— 提交到仓库：通用能力、schema、默认值
schema_version: 1
tools:
  ffmpeg:
    command: ${FFMPEG_BIN:-ffmpeg}
    cwd: ${AGENT_PROJECT_ROOT:-.}
    timeout: 30s
    env:
      - TMPDIR=${AGENT_TMP:-/tmp}
```

```markdown
# tools.local.md —— 不提交：本机绝对路径、端口、调试开关
ffmpeg:
  command: /opt/homebrew/bin/ffmpeg
```

解析优先级建议固定为：

> 命令行参数 > 当前 shell 环境变量 > `tools.local.md` > `tools.md` > 内置默认值

这样本地差异被隔离在 `tools.local.md`，仓库里的 `tools.md` 保持通用。平台差异可以在 `tools.md` 里写分支：

```markdown
ffmpeg:
  command:
    darwin: /opt/homebrew/bin/ffmpeg
    linux: /usr/bin/ffmpeg
    windows: C:\tools\ffmpeg.exe
```

但更推荐把平台分支也放进 `tools.local.md`，让主配置不感知宿主机。

## 四、踩坑点

1. **`~` 不会自动展开**  
   如果工具不是通过 shell 启动，而是直接 exec，`~` 不会被解析。统一用 `$HOME` 或绝对路径。

2. **相对 cwd 是噩梦**  
   `cwd: ./scripts` 到底相对 Agent 进程启动目录，还是相对 `tools.md` 所在目录？不同实现不一样。建议显式使用项目根变量：`cwd: ${AGENT_ROOT}/workdir`，并在实际调用前展开。

3. **MCP 子进程不一定继承环境变量**  
   不要假设 stdio MCP server 能读到 Agent 的环境。需要在 `tools.md` 中显式声明 `env`，把需要的变量传进去，比如 `HOME`、`PATH`、`TMPDIR`、`PYTHONPATH`。

4. **密钥泄漏到 `tools.local.md`**  
   `tools.local.md` 如果不提交还好，但一旦复制到容器或打包进镜像，密钥就暴露。密钥只允许引用变量名，值本身放 `.env.local` 或系统环境变量。

5. **端口和超时差异**  
   本地调试端口 8080 可能很顺，部署环境里 8080 已被占用。可在 `tools.local.md` 里覆盖端口，并在预检脚本中检查占用。

## 五、可复用建议

- 仓库里提交 `tools.example.md` 和 `tools.md`，`tools.local.md` 加入 `.gitignore`。  
- `tools.md` 保持声明式，不要写复杂条件逻辑；逻辑放到 wrapper 脚本里。  
- 给 `tools.md` 加 `schema_version`，升级结构时能做兼容校验。  
- 做一个 `agent tools preflight --env local` 之类的预检命令，检查二进制是否存在、env 值是否为空、端口是否冲突。  
- 在 macOS / Linux / Windows 三个环境至少各跑一次同一个 Agent 的 smoke test，环境差异会提前暴露。  
- 工具名和 MCP 服务名尽量避免重名或别名不一致，否则排查时很难定位哪个配置生效。

## 六、总结

`tools.md` 不是一份文档，而是一道边界：边界内是 Agent 能依赖的稳定能力，边界外是每台机器自己的脾气。把本地路径、平台分支、环境变量和密钥都收拢到这一层，Agent 才能真正从“在我机器上能跑”变成“在别人的机器上也能跑”。

配置越克制，Agent 越可复用。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/37dfe5e55b1c39b5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/5a3417a40583c7c8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/537ff7f2670a64c0.png)

