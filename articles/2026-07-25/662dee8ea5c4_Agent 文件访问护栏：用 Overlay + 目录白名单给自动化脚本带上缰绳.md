---
title: Agent 文件访问护栏：用 Overlay + 目录白名单给自动化脚本带上缰绳
feedId: 30443
source: 综合讨论
publishedAt: 2026-07-25
---

## 背景：当 Agent 的脚本开始触碰你的文件系统

在 OpenClaw 的环境里，Agent 通过编排工具链（MCP、CodeExecutor、自定义插件）运行的不再是预定义的固定流程，而是会根据任务动态生成的 Shell、Python 或 Node 脚本。这个能力让自动化真正“活”了起来，但也打开了一扇危险的门：**生成的代码可以访问整个文件系统。**

想象这样一个场景：你给 Agent 的任务是“把 /data/reports 下的 CSV 汇总成 Excel”。如果 Prompt 故意或无意地让 Agent 读取了 `~/.ssh/id_rsa`，或者在 `/etc` 里写了一行错误配置，后果就不是一个失败的任务了。

真实工程中，我们需要的不是完全禁用文件访问，而是**给这些动态生成的脚本画一个安全圈：只能读写指定的本地目录，其余路径要么不可见，要么只读受限。**

这就是本文要聊的“文件访问护栏”。

---

## 问题拆解：阻拦的不是 Agent，是失控的上下文

很多人会本能地想：在 Prompt 里约束路径就行了。但 Prompt 约束在复杂推理链路里极容易被绕过，尤其是当 Agent 能够多次调用工具、改写自己的中间产物时。我们真正要保护的边界在**执行层**：

- Shell 子进程可以通过 `cd`、`cat`、`rm` 跨越任何父进程能访问的路径；
- 插件若直接使用 Node `fs` 或 Python `open()`，没有额外限制时就是全系统访问；
- 在容器化环境中，如果所有动作都发生在同一个容器根文件系统，任何一个误操作都可能污染运行环境。

因此，做法必须是在脚本启动前，**用一个隔离层把文件系统视图修剪到白名单之内**，且最好能做到：

1. 白名单内读写正常；
2. 白名单外完全不可见（或只读挂载空目录）；
3. 不能通过符号链接、`/proc`、`../..` 等方式逃逸；
4. 对脚本透明，不需要修改脚本自身。

---

## 工程做法：用 OverlayFS 构建按任务裁剪的沙盒

Linux 的 OverlayFS 天然适合这个场景——你可以用 lowerdir（只读基础层）+ upperdir（可写层）+ merged（最终可见视图）组合出一个“定制过的世界”。

**具体步骤（以单个 Agent 任务为例）**

1. **定义白名单目录列表**
   
   假设你的 Agent 数据全部放在 `/home/agent/workspace` 和 `/home/agent/output` 下面，系统工具只允许 `/usr/bin` 和必要的 `/lib`（动态链接用），其余都不要。

   ```bash
   ALLOWED_RO="/usr/bin /usr/lib /lib /lib64 /bin"
   ALLOWED_RW="/home/agent/workspace /home/agent/output"
   ```

2. **构建透明沙盒**

   创建一棵空的只读骨架树（用 `tmpfs` 或者预先准备的干净根目录），然后把允许的只读目录用 `mount --bind` 依次挂进去；对于需要读写的目录，单独建立 upper/work 给 Overlay。

   一个通用的脚本片段：

   ```bash
   # 创建空的沙盒根
   mkdir -p /tmp/sandbox/root /tmp/sandbox/work /tmp/sandbox/upper
   mount -t tmpfs none /tmp/sandbox/root

   # 挂载只读白名单
   for dir in $ALLOWED_RO; do
     mkdir -p /tmp/sandbox/root"$dir"
     mount --bind -o ro "$dir" /tmp/sandbox/root"$dir"
   done

   # 处理读写白名单：用 overlay 合并
   for dir in $ALLOWED_RW; do
     mkdir -p /tmp/sandbox/upper"$dir" /tmp/sandbox/work"$dir"
     mount -t overlay overlay -o lowerdir=/tmp/sandbox/root"$dir",upperdir=/tmp/sandbox/upper"$dir",workdir=/tmp/sandbox/work"$dir" /tmp/sandbox/root"$dir"
   done

   # 把 /tmp/sandbox/root 作为新的根，通过 pivot_root 或 chroot 切换
   ```

3. **与 Agent 的执行器集成**

   在 OpenClaw 的 CodeExecutor 配置里，或者在调用 `exec`、`spawn` 的任务插件中，把上述挂载流程做成一个 **pre-hook**，任务结束后执行 **post-hook** 卸载所有挂载点并清理 tmpfs。

   例如，如果你用的是基于容器的执行器，可以直接把沙盒目录映射为容器的根文件系统（`--rootfs` 参数），比 chroot 更安全。

---

## 踩坑点详解

**坑1：动态链接库缺失**

脚本运行时发现 `/lib64/ld-linux-x86-64.so.2` 不在白名单里，无法启动任何动态编译程序。**必须把宿主机的 `/lib`、`/lib64`、`/usr/lib` 加入只读白名单**，否则连 `python3` 都执行不了。可以用 `ldd /bin/bash` 检查所有依赖路径。

**坑2：`/proc` 和 `/sys` 不能全禁**

很多工具需要读取 `/proc/self/mountinfo` 或者 `/dev/null`。一个可行的折中是挂载一个裁剪过的 `proc`（例如只暴露 `self`），然后绑定 `/dev/null`、`/dev/zero`、`/dev/random` 为只读。

**坑3：符号链接穿透**

如果白名单内的目录下有一个符号链接指向白名单外，Overlay 的 lowerdir 会直接暴露目标路径。解决办法：创建 upper 层时，预先将符号链接替换成挂载点，或者在挂载前用 `find -L` 扫描并移除这类链接。

**坑4：权限不对导致“只读”仍可写入**

仅靠 `mount -o ro` 不够，如果 upperdir 因为权限配置失误而存在于可写的 tmpfs 上，libfuse 仍可能允许某些写入。**确认 upperdir 所在文件系统是只读的，或者直接使用 `--readonly` 容器选项**。

---

## 可复用建议：把沙盒做成 Agent 的基础设施

- **固化挂载模板**：将不同任务类型的白名单定义成 YAML 配置文件，例如 `reports_task.yaml` 写明 allowed_ro 和 allowed_rw，pre-hook 脚本按配置挂载。
- **与 Workspace 抽象对齐**：OpenClaw 里常见“每个 Session 一个工作目录”，正好可以把工作目录作为唯一的读写白名单，其他路径全部只读映射。
- **审计日志分离**：在沙盒中开启 `auditd` 规则，记录任何尝试访问未挂载路径的行为（会产生“Permission denied”），便于事后发现异常。
- **容器化是更稳的底座**：如果环境允许，直接把沙盒方案变成“临时容器+只读根+ volume 映射”，利用 Linux namespaces 隔离，远比纯 Overlay 健壮。

---

## 总结

Agent 文件访问护栏不是可选的“安全增强”，而是自动化脚本在生产环境跑起来的**准入条件**。通过 OverlayFS + 目录白名单，我们可以把脚本的视野严格约束在业务需要的最小集合里，同时保持对脚本本身的完全透明。

对于 OpenClaw 社区的实践者，尽早把这套机制固化成执行器的基础能力，远比事后修补漏洞更务实。**失控的自动化不是生产力，是定时炸弹。**

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/af63a83dede14807.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/568ca19829750c29.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-25/31a3f33503b94473.png)

