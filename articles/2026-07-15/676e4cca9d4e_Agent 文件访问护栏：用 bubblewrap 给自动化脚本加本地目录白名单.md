---
title: Agent 文件访问护栏：用 bubblewrap 给自动化脚本加本地目录白名单
feedId: 29179
source: 综合讨论
publishedAt: 2026-07-15
---

## 背景：脚本执行权越大，踩坑越深

在 OpenClaw 的日常使用中，很多场景需要 Agent 调用本地自动化脚本——从日常文件整理、日志清洗到临时生成测试数据，这些脚本通常由大模型生成或用户预先编写。多数情况下，开发者为了快速打通链路，会把 Agent 的工具权限直接映射到宿主系统的 `fs` 模块或 `/bin/bash`，这就意味着：**一次 prompt 生成的不完整脚本，有机会翻遍你的整个 `$HOME`**。

即使你不担心恶意代码，大模型生成的内容也时有“路径拼错”、“删错临时目录”、“不小心覆盖 `~/.ssh/config`”这类工程事故。真正需要的是一个「文件访问护栏」——让自动化脚本只能碰我们显式许可的目录，其它路径对脚本而言根本不存在。

## 问题拆解：不是拒绝访问，是看不见

许多人的第一反应是“写个包装函数，检查每个路径再调用”。但如果你允许脚本直接执行系统命令（比如 `python script.py` 或 `bash run.sh`），那么进程级别的文件访问你拦不住——除非打内核补丁或使用 LD_PRELOAD 劫持，但这两种方式破坏性太大且不可维护。

更可靠的方式是**在进程启动时就限制它的文件系统视图**。Linux 内核的 namespace 可以把挂载点隔离出来，配合 bind mount 构建一个“白名单沙箱”。比如只把 `/data/project`、`/tmp` 和必要的系统库暴露给自动化脚本，其他目录（`/home`、`/etc` 等）在沙箱内就是空或不存在。这样即使脚本尝试访问敏感文件，也会因为路径不存在直接失败，无需在运行时反复过滤。

## 落地步骤

以下用 `bubblewrap`（简称 `bwrap`）实现，它是 Flatpak 项目的产物，不需要 root，且能在大多数现代 Linux 发行版上直接使用。如果你用 macOS，可考虑 `sandbox-exec`，思路类似。

### 1. 安装 bubblewrap
多数发行版已打包：
```bash
sudo apt install bubblewrap  # Debian/Ubuntu
sudo dnf install bubblewrap  # Fedora
```

### 2. 设计最小文件系统视图
列出脚本运行必需的系统目录和白名单业务目录。典型最小清单：
- `/usr` (只读)：解释器、系统库
- `/lib` `/lib64` (只读)：动态链接器
- `/bin` (只读)：shell、基础命令
- `/etc/ld.so.cache` `/etc/alternatives` 等（视情况）
- `/proc` (只挂载 proc)：`/proc/self/exe` 等
- `/dev` (最小设备)：`/dev/null`、`/dev/random`、`/dev/urandom`
- 业务目录(读写)：如 `/srv/agent-workspace`
- `/tmp` (独立 tmpfs, 避免污染宿主 `/tmp`)

### 3. 编写通用沙箱包装器
创建 `agent-sandbox` 脚本：
```bash
#!/bin/bash
RW_DIR="${RW_DIR:-/srv/agent-workspace}"
CMD="$@"

exec bwrap \
  --ro-bind /usr /usr \
  --ro-bind /lib /lib \
  --ro-bind /lib64 /lib64 \
  --ro-bind /bin /bin \
  --ro-bind /etc/ld.so.cache /etc/ld.so.cache \
  --proc /proc \
  --dev /dev \
  --tmpfs /tmp \
  --bind "$RW_DIR" "$RW_DIR" \
  --unshare-all \
  --die-with-parent \
  -- "$CMD"
```
说明：
- `--ro-bind` 将宿主路径只读挂载进沙箱，脚本看不到其它。
- `--bind` 将业务目录以读写方式暴露。
- `--unshare-all` 同时隔离网络、PID 等，进一步收窄权限。
- `--tmpfs /tmp` 让脚本的临时文件与宿主 `/tmp` 隔离。
- 如需多个白名单目录，追加 `--bind /another/path /another/path`。

### 4. 在 OpenClaw 工具中集成
假设你的 Agent 通过一个“本地执行器”工具运行脚本，只需把执行方式改为：
```python
subprocess.run(["agent-sandbox", "python", "script.py"], ...)
```
如果是 MCP server 提供的文件命令，可以修改其 shell 执行入口，统一经过 `agent-sandbox`。白名单目录可通过环境变量 `RW_DIR` 注入，实现动态配置。

## 踩坑记录

**基础系统文件遗漏**
第一次运行很可能会报 `error while loading shared libraries`。这时用 `strace -f -e openat bwrap ...` 检查缺少哪些 `.so` 文件，然后追加 `--ro-bind` 把对应目录挂入。通常补上 `/etc/ld.so.cache` 和 `/etc/ld.so.conf` 能解决大部分问题。

**符号链接逃逸**
如果业务目录内部有指向 `/home` 的符号链接，沙箱内访问该符号链接会失败（因为 `/home` 未挂载，安全）。但如果有人刻意在业务目录里创建指向外部的链接，仍然无法真正“逃逸”，因为目标路径不可见。重点是不挂载整个 `/`。

**`/dev` 设备不当**
脚本可能依赖 `/dev/null`，但若未挂载 `/dev`，`> /dev/null` 会创建普通文件。沙箱提供最小 `/dev` 即可；bwrap 的 `--dev /dev` 会创建新的 devtmpfs，安全且够用。

**性能与兼容性**
`bwrap` 本身几乎零性能损耗。在容器环境下需要内核开启 user namespace，大部分云主机默认启用。如果你的环境禁用了 user namespace（`/proc/sys/kernel/unprivileged_userns_clone` 为 0），需要用 root 执行或开启该参数。

**临时目录泄密**
如果直接把宿主 `/tmp` bind 进去，脚本可能读出其他进程的临时文件。务必用 `--tmpfs /tmp` 新建独立 tmpfs。

## 可复用建议

**抽象白名单配置**
将白名单做成单独配置文件，例如 `/etc/agent-whitelist.json`，在 wrapper 中解析。这样不同任务可以使用不同沙箱实例。OpenClaw 可以在 Agent 上下文里注入当前可用的白名单目录列表，让模型知道“你只能写这些地方”。

**配合只读挂载实现分层授权**
对于模型生成的脚本，可把业务目录也分为“只读资源区”和“读写工作区”，例如：
- `--ro-bind /data/inputs /data/inputs`
- `--bind /data/outputs /data/outputs`

这样即使脚本逻辑出错，也不会污染输入数据集。

**在 CI/测试环境中复用**
同样的 `agent-sandbox` 脚本可以用在本地测试：写单元测试时直接用 bwrap 包裹被测命令，断言它没有超出白名单访问。

**macOS 替代方案**
macOS 下可用 `sandbox-exec` 编写类似约束的 profile，但语法较晦涩。轻量场景可直接用 Docker（`-v` 仅挂载白名单目录）或使用系统自带的 App Sandbox。不过 Docker 有额外 overhead，只建议已有容器化底座时采用。

## 总结

给 Agent 的自动化脚本加本地目录白名单不是过度设计，而是工程上必要的安全基线。借助 `bubblewrap` 这一轻量级内核特性，我们可以用几十行 shell 把“大模型写的脚本”关进一个只有白名单目录的虚拟文件系统，彻底杜绝误操作和敏感文件泄露。这套方案部署成本低、对业务透明，并且能与 OpenClaw 的工具链自然集成。

把权限“看不见”比“拦住它”更难，但也更可靠——因为脚本甚至不知道有其他路径存在。这正是文件访问护栏的核心哲学，也值得你在下一个 Agent 项目中立刻应用。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-15/61ab55f6aad94277.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-15/f61552da614388f3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-15/bb3cb3f4808dcf88.png)

