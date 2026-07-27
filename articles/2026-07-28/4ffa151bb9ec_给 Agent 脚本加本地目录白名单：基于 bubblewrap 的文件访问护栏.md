---
title: 给 Agent 脚本加本地目录白名单：基于 bubblewrap 的文件访问护栏实践
feedId: 30718
source: 综合讨论
publishedAt: 2026-07-28
---

## 背景

在 OpenClaw 或任何允许 Agent 调用本地脚本的系统中，一个常见场景是：为了让 Agent 完成特定任务，我们需要给它执行 Shell 命令的权限。可能是运行一段 Python 做数据处理，也可能是调用 `ffmpeg` 转码，或是执行用户自定义的自动化脚本。

一旦给了执行权限，风险随之而来。一条错误命令可能误删文件、泄露 `~/.ssh` 下的密钥，或者被注入恶意指令访问不该碰的目录。传统的 sudo 或 chroot 过于笨重，Docker 又引入了额外的依赖与资源开销。我们需要一种轻量的、非特权用户的目录级隔离方案，给 Agent 的执行环境加上“本地目录白名单”。

## 问题拆解

目标很明确：  
- Agent 只能访问我们指定的目录（如项目目录、临时工作目录）。  
- 不能访问家目录、系统配置、其他用户数据。  
- 不需要 root 权限，最好作为普通用户运行。  
- 对现有脚本改动最小，能直接套用在 OpenClaw 的 MCP 或插件调用中。

Linux 内核的命名空间为我们提供了可能。`bubblewrap`（命令 `bwrap`）正是基于用户命名空间、挂载命名空间等特性的封装工具，无需 root 即可创建受限的文件系统视图，非常适合这一场景。

## 做法：用 bubblewrap 构建白名单沙箱

### 1. 安装 bubblewrap

多数发行版已内置或可从包管理器安装：

```bash
# Debian/Ubuntu
sudo apt install bubblewrap
# Fedora
sudo dnf install bubblewrap
# Arch
sudo pacman -S bubblewrap
```

确认可用：

```bash
bwrap --version
```

### 2. 确定白名单目录与挂载策略

假设我们的项目位于 `/home/user/project`，需要读写权限；Agent 还可能产生临时文件，需要一个独立的 `/tmp`；另外脚本依赖系统基础库，不能完全隔离 `/usr`，但可以只读挂载必要的部分（如 `/usr/lib`、`/usr/bin`），并屏蔽 `/home` 下其他路径。

一个典型的策略：  
- 以项目根目录为新文件系统的根（或挂载到 `/app`）。  
- 挂载一个空目录作为 `/tmp`，避免使用宿主机 /tmp。  
- 只读挂载 `/usr`（或细分），确保动态库、解释器可访问。  
- 创建新的 `/proc`、`/dev` 最小设备集。  
- 不挂载 `/home`，不暴露用户个人文件。

### 3. 编写包装脚本

创建 `sandbox.sh`：

```bash
#!/bin/bash
set -e

SANDBOX_ROOT="/home/user/project"
WORKDIR="/app"
TMPDIR=$(mktemp -d /tmp/agent-tmp-XXXXXX)

exec bwrap \
  --unshare-all \
  --share-net \
  --die-with-parent \
  --ro-bind /usr /usr \
  --ro-bind /etc /etc \
  --proc /proc \
  --dev /dev \
  --tmpfs /tmp \
  --bind "$TMPDIR" /tmp \
  --bind "$SANDBOX_ROOT" "$WORKDIR" \
  --chdir "$WORKDIR" \
  --clearenv \
  --setenv PATH /usr/bin:/bin \
  --setenv HOME /tmp \
  "$@"
```

关键参数解释：  
- `--unshare-all`：创建新的用户和挂载命名空间，关闭不必要的共享。  
- `--share-net`：保留网络访问，如果不需要可以改为 `--unshare-net`。  
- `--die-with-parent`：当包装脚本退出时，bwrap 也终止，防止残留进程。  
- `--ro-bind /usr /usr`：只读挂载宿主机的 `/usr`，这样脚本可以调用 Python、ffmpeg 等。  
- `--bind "$SANDBOX_ROOT" "$WORKDIR"`：将项目目录以读写方式绑定到沙箱内的 `/app`。  
- `--tmpfs /tmp` + `--bind "$TMPDIR" /tmp`：先建立 tmpfs 再绑定一个干净的临时目录，确保 `/tmp` 干净且独立于宿主机。  
- `--clearenv` 加 `--setenv`：清除宿主机环境变量，仅传递必要值，防止无意泄露配置信息。

### 4. 集成到 Agent 执行命令

在 OpenClaw 的工具配置中，将原本的命令替换为：

```bash
/path/to/sandbox.sh python3 my_script.py
```

如果需要传递参数，正常追加即可。这样，Agent 对文件系统的视图被严格限制在 `/app`（即项目目录）和干净的 `/tmp` 内，无法访问 `/home` 下的其他目录，`/usr` 也只是只读。

## 实际踩坑与应对

### 动态库缺失

有些脚本依赖的库并不仅仅在 `/usr/lib` 下，可能在 `/lib`、`/lib64` 或 `/usr/local/lib`。如果运行时报 `error while loading shared libraries`，需要额外挂载这些路径：

```bash
--ro-bind /lib /lib
--ro-bind /lib64 /lib64
--ro-bind /usr/local/lib /usr/local/lib
```

一个快速定位缺失库的方法：  
```bash
ldd $(which python3)
```
查看所有链接路径，对照挂载列表。

### /etc 中的配置文件

很多程序会读取 `/etc/resolv.conf`（DNS）、`/etc/ssl/certs`（TLS）等。如果沙箱内网络请求失败或证书错误，要考虑将相关文件以只读方式绑定进去，而不是整体挂载 `/etc`。更精细的做法是只挂载必要文件：

```bash
--ro-bind /etc/resolv.conf /etc/resolv.conf
--ro-bind /etc/ssl/certs /etc/ssl/certs
```

这能进一步缩小文件暴露面。

### 用户身份的混淆

bwrap 在沙箱内默认以当前用户身份运行，因此文件权限与宿主机一致。如果脚本在沙箱内创建文件，其在宿主机上的属主仍是该用户，这符合预期。但要注意，`$HOME` 被重设为 `/tmp`，某些程序可能因此找不到 `.config` 等配置，若确实需要可以挂载一个专门的配置目录。

### 性能与资源限制

bwrap 本身几乎无性能损耗。若需要额外限制内存或 CPU，可以结合 `systemd-run --user` 或 `ulimit`，但这超出了文件访问护栏的范畴。对一般 Agent 任务，默认命名空间隔离已足够。

## 可复用建议

1. **最小权限原则**：只挂载脚本真正需要访问的路径。白名单目录可以设置为只读，除非明确需要写入。  
2. **使用临时目录**：每次调用创建独立的临时目录，并在脚本结束后清理。可以在包装脚本尾部添加 `trap "rm -rf $TMPDIR" EXIT`。  
3. **环境变量剥离**：务必 `--clearenv`，然后显式设置 PATH、HOME 等。避免将宿主机的 token、密钥等环境变量透传到沙箱。  
4. **结合 MCP 的配置模板**：将沙箱包装脚本作为 OpenClaw 的 `command_prefix` 或 MCP 服务器的启动参数，无需修改每个工具调用。  
5. **测试覆盖**：编写一个“逃逸测试”脚本，尝试读取 `/etc/shadow`、写入 `/home/user/.ssh/authorized_keys`，验证沙箱是否真正拦截。

## 总结

利用 `bubblewrap` 为 Agent 执行的本地脚本添加目录白名单，是一种投入成本低、效果立竿见影的工程化安全实践。它不依赖 root、不引入容器运行时，几十行脚本就能将文件访问锁定在指定目录内，有效防止误操作或恶意命令带来的数据泄露与破坏。

在 OpenClaw 的自动化工作流中，将文件访问护栏作为默认的安全层，既能保持 Agent 的灵活性，又能将风险控制在可接受范围内。建议团队将类似的包装脚本固化到部署流程中，让每一次 Agent 命令执行都默认为受限环境。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/defd703ac437b42d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/e71fc23b95d2d662.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-28/9f7973a7fda46b9d.png)

