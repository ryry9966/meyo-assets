---
title: 深度解析 OpenClaw 的沙箱安全模型：为什么你的 Agent 不会误删宿主机文件
feedId: 32152
source: 综合讨论
publishedAt: 2026-08-08
---

# 深度解析 OpenClaw 的沙箱安全模型：为什么你的 Agent 不会误删宿主机文件

## 背景：当自动化 Agent 拥有文件写入权限

在构建基于 OpenClaw 的任务编排与自动化体系时，一个无法回避的问题就是安全边界。尤其是当你允许 Agent 调用系统命令、安装依赖、生成临时文件，甚至通过 MCP FileSystem Server 读取或写入本地目录时，一个常见焦虑是——“它会不会一不小心 `rm -rf /` 把项目源码全删了？”

这类担心并非多余。传统自动化脚本如果没有做好路径限制，确实可能因为变量未初始化、通配符展开错误而“删库跑路”。但在 OpenClaw 的容器化执行模型里，这种风险被多层沙箱机制从设计上消除了。

本文将围绕 OpenClaw 执行环境的隔离策略，展示 Agent 为什么不会（即使“想”也做不到）破坏宿主机文件系统。

---

## 问题：为什么普通进程无法保证文件安全？

一个直接运行在宿主机上的 Agent 进程，如果以当前用户权限启动，它默认可以访问所有你拥有权限的文件。即使你通过 `chroot` 或者虚拟环境限制工作目录，进程仍然可能通过以下方式逃逸：

- 绝对路径访问：`rm -rf /home/user/projects`
- 符号链接穿越：在可写目录创建指向外部敏感路径的软链接
- `/proc` 或 `/sys` 暴露宿主机信息
- 挂载点逃逸

因此，文件安全的关键不在于“教 Agent 别乱删”，而在于**让它只能看见并写入你允许的部分**。这是 OpenClaw 容器化 Agent 运行时的核心安全理念。

---

## 实践：OpenClaw 如何实现文件系统隔离

OpenClaw 默认采用 Docker 作为每个 Agent 的执行沙箱。以下是一个真实可复现的配置过程，展示如何让 Agent 只能写入指定的工作区，而项目源码、系统文件一概不可修改。

### 1. 容器编排中实施只读根文件系统

在 `docker-compose.yml` 中，为 Agent 服务设置 `read_only: true`，仅对必要目录挂载可写卷：

```yaml
services:
  openclaw-agent:
    image: openclaw/agent:latest
    read_only: true
    volumes:
      - ./workspace:/workspace:rw
      - ./scripts:/scripts:ro
      - /tmp/agent-tmp:/tmp:rw
    environment:
      - WORKSPACE_DIR=/workspace
```

关键点：
- 根文件系统（`/`）只读，Agent 无法修改任何系统文件或配置
- `/workspace` 是唯一允许读写的业务数据目录
- 项目脚本目录 `/scripts` 只读挂载，即使 Agent 尝试删除也会收到 `Read-only file system` 错误

### 2. 通过 MCP 工具限制文件访问边界

若使用 MCP FileSystem Server 暴露文件操作接口，可以在 OpenClaw 的 Agent 配置中限定允许的路径列表：

```yaml
tools:
  - name: filesystem
    type: mcp
    config:
      allowedDirectories:
        - /workspace
        - /tmp
```

这样一来，即使 Agent 刻意构造路径穿越字符串（例如 `../../../etc/passwd`），MCP 服务端也会在路径解析后拒绝操作，因为解析出的真实路径不在白名单中。

### 3. 禁止容器特权与限制 Capabilities

OpenClaw 编排文件中建议明确关闭特权模式并裁剪 Linux capabilities，防止 Agent 通过挂载新文件系统或修改设备来绕过只读限制：

```yaml
privileged: false
cap_drop:
  - ALL
cap_add:
  - DAC_OVERRIDE  # 如非必要也不加
security_opt:
  - no-new-privileges:true
```

经过以上配置，Agent 在容器内部即使执行 `sudo rm -rf /`（假设容器内存在 sudo）也只能删除存在于 `/workspace` 或 `/tmp` 下的文件，因为这些是唯一挂载的可写层。根文件系统的修改会被联合文件系统拦截，只读挂载的目录直接返回错误。

### 4. 验证隔离效果

进入 Agent 容器，尝试删除关键目录：

```bash
docker exec -it openclaw-agent bash
rm -rf /scripts       # 报错：Read-only file system
rm -rf /etc           # 报错：Read-only file system
rm -rf /workspace/*   # 成功，但只影响工作区
```

宿主机上的源码、配置不受任何影响。

---

## 踩坑记录

在实践中，有几个细节容易让沙箱出现破口：

- **挂载了宿主机目录为可写**：有时为了调试方便，把整个项目目录挂载为 `rw`，这直接暴露了宿主机文件。一定要区分哪些目录需要 Agent 写入，其余一律 `ro`。
- **忘记限制 `/tmp`**：很多程序会在 `/tmp` 下创建临时文件，如果挂载为宿主机 `/tmp`，可能引发信息泄露或资源耗尽。建议使用独立卷或 `tmpfs`。
- **Agent 工具链允许执行任意 Shell 命令但未限制工作目录**：若给予 Agent 开放的命令执行权限，它仍可能通过 `cd /scripts && rm -rf .` 删除只读挂载的内容吗？答案是不能，因为文件系统只读。但可能会造成心理恐慌，应该在日志中记录命令执行上下文。
- **使用非容器化运行模式**：如果开启了“本地执行”模式（非容器化），上述所有文件系统隔离都会失效。生产环境务必使用容器化沙箱。

---

## 可复用的安全建议

基于这次实践，你可以将以下清单作为 OpenClaw Agent 部署的安全基线：

1. **永远以容器方式运行 Agent**，避免本地执行
2. **根文件系统只读**，仅对特定工作区授予读写权限
3. **项目源码、配置文件以只读卷形式挂载**
4. **通过 MCP 工具层再进行路径白名单校验**，双重防护
5. **丢弃所有 Linux capabilities**，禁止特权模式，设置 `no-new-privileges`
6. **启用日志审计**，记录 Agent 的所有文件操作命令，便于回溯
7. **定期检查容器逃逸相关 CVE**，保持 Docker Runtime 更新

---

## 总结

OpenClaw 的安全模型并不是简单寄希望于 Agent 的“靠谱”，而是通过容器化隔离、只读根文件系统、细粒度挂载与工具层路径控制，构成了一个纵深防御体系。在这个体系下，Agent 的误操作半径被严格限制在可写工作区之内，宿主机文件系统对 Agent 而言是物理不可达的。

理解了这一层，你就可以放心让 Agent 执行自动化的文件处理、代码生成、报告输出等任务，而无需担心一觉醒来项目目录不翼而飞。安全不是口号，是设计出来的边界。

---

