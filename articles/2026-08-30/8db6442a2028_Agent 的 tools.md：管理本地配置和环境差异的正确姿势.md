---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 35277
source: 综合讨论
publishedAt: 2026-08-30
---

# Agent 的 tools.md：管理本地配置和环境差异的正确姿势

## 背景

OpenClaw/Agent 类项目经常把工具描述、MCP 端点、插件端口、数据目录写进提示词或代码里。换一台机器、换一个容器、换一个 CI runner，第一反应往往是：为什么找不到 `ffmpeg`、为什么 MCP 连不上、为什么 `sed` 报错。

`tools.md` 不是给模型看的“能力说明书”，而更像一份机器可读的本地环境事实表：OS、Shell、路径、运行时、服务地址、可选工具和降级命令。写得克制，它能让 Agent 少做错误假设；写得太满，又会变成没人维护的过期文档。

## 问题

典型症状：

- 在 macOS 上能跑，在 Linux 容器里 `sed -i ''` 直接报错；
- 本地个人路径 `/Users/xx/code/foo` 被写死，CI 上永远找不到；
- token、API Key 或内网地址被塞进 `tools.md`，导致仓库泄露；
- MCP/插件端口在 Docker 里写 `localhost`，实际要连 `host.docker.internal`；
- 工具缺失时 Agent 直接失败，不会降级或跳过。

这些不是模型能力问题，而是环境信息没有结构化、没有边界。

## 做法/步骤

### 1. 只记录事实，不写脚本

`tools.md` 适合放“是什么”，不适合放“怎么做”。命令模板可以有，但长脚本应放 `scripts/`，`tools.md` 只引用入口。

### 2. 固定一个最小骨架

```markdown
# tools.md

## os
os: linux
arch: x86_64
shell: bash

## paths
repo_root: ${REPO_ROOT:-.}
data_dir: ${DATA_DIR:-./data}
tmp_dir: ${TMPDIR:-/tmp}

## runtime
python: python3.12
node: node20
ffmpeg: ffmpeg

## services
mcp_gateway: ${MCP_GATEWAY:-http://127.0.0.1:8787}
browser_debug: ${BROWSER_DEBUG:-http://127.0.0.1:9222}
plugin_registry: ${PLUGIN_REGISTRY:-http://127.0.0.1:7788}

## commands
sed_inplace: sed -i ''
archive: tar -czf
checksum: shasum -a 256
```

关键不是内容多全，而是每个值都能被覆盖：优先读环境变量，`tools.md` 只是默认值。

### 3. 让 Agent 启动时做一次“环境校验”

不要假设 `tools.md` 写什么就有什么。可在启动或进入任务前执行：

```bash
command -v python3.12 >/dev/null 2>&1 || echo "missing: python3.12"
curl -fsS --max-time 2 "$MCP_GATEWAY/health" >/dev/null 2>&1 || echo "unreachable: MCP_GATEWAY"
```

校验结果回填给 Agent，比让它自己撞错更稳。

### 4. 区分 required 与 optional

```markdown
required_tools: git, node20, python3.12
optional_tools: ffmpeg, jq, playwright
```

Agent 看到 `optional` 缺失时，可以提示“跳过视频处理”或走降级路径，而不是硬跑。

### 5. 环境差异单独成节

把平台相关命令集中：

```markdown
## platform_delta
linux_sed_inplace: sed -i
darwin_sed_inplace: sed -i ''
linux_checksum: sha256sum
darwin_checksum: shasum -a 256
windows_archive: tar -a -c -f
```

这样模型不用猜当前平台，按 `tools.md` 里的键取值即可。

## 踩坑点

1. **不要写绝对路径。** 所有本地目录都用 `repo_root + 相对路径`，个人路径通过环境变量注入。
2. **不要存 secrets。** API Key/token 只写变量名，如 `${OPENAI_API_KEY}` 或 `${BROWSER_WS_TOKEN}`，值从 Secret 管理或 `.env` 注入，且 `.env` 不进仓库。
3. **不要把 `tools.md` 写成“提示词扩写”。** 每加一段长解释，Agent 的有效注意力就被稀释。工具描述越短，执行越稳。
4. **不要忽略 Windows/PowerShell。** 即使暂时只在 Linux 跑，也保留一张 `platform_delta` 表，后续接入 Windows runner 时不会到处 `if`。
5. **localhost 在容器里不是宿主机。** 若 MCP 或插件跑在宿主机，容器内 Agent 要连 `host.docker.internal` 或显式 `HOST_IP`。
6. **避免重复维护。** 尽量由探测脚本生成 `tools.md`；手写版本只覆盖探测不到的部分。

## 可复用建议

- 建立一个 `scripts/env_probe.sh`，输出 JSON 或 YAML，再模板化成 `tools.md`。
- `tools.md` 纳入版本控制，但只包含非敏感、可公开的环境事实；个人机差异通过 `.env` 注入。
- 在 `/health` 或启动日志里输出当前读取到的 `tools.md` 关键值，方便排障。
- 对 MCP/插件端口，统一使用 `127.0.0.1` 作为默认值，容器场景用 `HOST_IP` 覆盖，别写死容器名。
- 给 `tools.md` 设一个“最后自动探测时间”，超过 24 或 48 小时提示重新生成，减少过期文档。

## 总结

`tools.md` 的作用是把“我的机器能跑”变成“当前执行环境已知”。它不需要写得漂亮，但必须能覆盖真实差异：路径、二进制、平台命令、可选能力、服务地址。把 secrets 挡在外面，把平台差异显式化，让 Agent 先校验、再执行，比不断加提示词更有效。

本地配置管理的目标不是消除差异，而是让差异在进入任务前被看见。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/0db10d587ce70ea9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/555df5111603f6c9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/e199ebbc6c8d9a7a.png)

