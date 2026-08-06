---
title: OpenClaw 的沙盒安全模型深度解析：为什么 Agent 真的不会误删你的文件
feedId: 31940
source: 综合讨论
publishedAt: 2026-08-07
---

## 背景：当 Agent 拥有文件系统权限时，我们在担心什么？

在 OpenClaw 的插件与自动化实践中，越来越多用户开始让 Agent 直接操作本地文件系统——批量重命名、生成报表、清理临时文件、甚至调用 MCP 工具读写配置。能力越强，焦虑越深：“万一它执行了 `rm -rf` 怎么办？”“如果提示词注入导致模型误判，会不会把项目目录清空？”

传统做法往往走两个极端：要么完全禁止 Agent 操作文件，把自动化锁死在沙箱外；要么全量信任，出了事再靠 Git 回滚。OpenClaw 在设计之初就内置了一套可配置的沙盒安全模型，允许用户在“可用性”与“安全性”之间精细调参。本文就拆解这套模型的工作原理、配置方法和真实踩坑经验。

## 问题本质：Agent 的文件操作风险到底来自哪里？

风险并非只来自“模型发疯”。更有可能是：
1. **提示词注入**：上游消息携带恶意指令，诱导 Agent 删除关键路径。
2. **路径拼接错误**：Agent 在生成命令时出现幻觉，把 `~/project` 写成 `/home/user/project` 并误操作。
3. **通配符展开失控**：模型输出的命令中包含了 `*`，被 Shell 展开后作用域超出预期。
4. **工具调用边界模糊**：MCP 插件提供了 `write_file` 能力，但没有约束可写目录。

OpenClaw 的安全模型正是针对这些路径做了多层防御，核心思想是：**不依赖模型的“自觉”，而是通过执行层的前置拦截，让危险操作根本无法抵达系统调用。**

## 做法：开启并细调 OpenClaw 沙盒安全模型

### 1. 文件系统访问白名单（Allowlist）

在 OpenClaw 的 Agent 配置中，所有文件读写工具默认被禁，必须显式声明可访问路径。

```yaml
# agent.yaml（示例片段）
tools:
  - name: file_read
    enabled: true
    sandbox:
      allowed_paths:
        - /home/user/projects/openclaw-workspace/**
        - /tmp/openclaw-sandbox/**
      allow_hidden_files: false
```

- `allowed_paths` 支持 glob 模式，`**` 表示递归所有子目录。未列入的路径完全不可见，即使 Agent 尝试 `cat /etc/passwd`，工具侧也会直接拒绝并记录审计日志。
- `allow_hidden_files: false` 可以防止 Agent 操作 `.git`、`.env` 等敏感隐藏文件，这是很多用户容易忽略的细节。

### 2. 操作类型黑名单（Denylist）

白名单只能限制“读不到”，但如果 Agent 在白名单路径内执行破坏性操作（如删除整个项目），仍会造成灾难。OpenClaw 引入了操作类型的声明式黑名单：

```yaml
sandbox:
  deny_operations:
    - recursive_delete
    - chmod 777
    - execute_shell_command
```

- `recursive_delete`：禁止任何形式的递归删除，哪怕路径在白名单内。
- `execute_shell_command`：默认禁用，防止 Agent 绕过工具层直接调用 Shell。如果你确实需要执行特定脚本，建议换成“受限命令通道”，只允许执行签名的脚本文件。

### 3. 写前确认（Write Gate）

对于高风险写操作（如删除、覆盖文件），可以开启写前确认机制：

```yaml
sandbox:
  write_gate:
    enabled: true
    require_user_approval_for:
      - delete
      - overwrite
      - rename
    timeout_seconds: 30
```

Agent 执行到这类操作时会暂停，并通过 UI 或消息通道推送审批请求，用户必须在 30 秒内确认，否则自动拒绝。这在不完全自主的场景（如本地调试）中非常实用。

### 4. 虚拟文件系统层（VFS Layer）——最易被低估的防护

OpenClaw 的 File Tool 实际上运行在一个虚拟文件系统层之上。它并不是直接调用 `os.write()`，而是通过一个文件系统代理：

- 所有路径在写入前会被规范化、解析符号链接，并再次与 `allowed_paths` 做对比。
- 写操作默认采用“写时复制”（Copy-on-Write）的安全模式——新建文件会先写入沙盒快照，只有操作完整成功后才会同步到物理磁盘。如果 Agent 在中途崩溃或触发超时，写入不会污染真实文件。

这个设计对于插件开发者尤为重要。如果你在 MCP 插件中封装了文件写入，建议也复用 OpenClaw 提供的 `SandboxedFileSystem` 接口，而不是自行调用 `fs` 模块。

## 踩坑实录：真实配置中容易犯的 3 个错误

1. **`allowed_paths` 配置过宽，且未禁用 `execute_shell_command`**  
   早期测试时，我把 `/home/user/**` 加入白名单，又忘了禁用 Shell 执行。Agent 在一次自动化清理任务中生成了一条包含 `find /home/user -name '*.tmp' -delete` 的命令，通过 Shell 工具执行后直接清理了我的文档目录。教训：**白名单尽量收敛到项目目录；永远不要同时开放宽白名单和 Shell 能力。**

2. **信任了隐藏文件保护，但没有禁止递归删除**  
   我勾选了 `allow_hidden_files: false`，以为 `.git` 安全了。但 Agent 执行了 `rm -rf openclaw-workspace`，因为那个目录不是隐藏文件，只是目录内包含隐藏子目录。当 `recursive_delete` 未禁止时，整个目录树还是被删光，包括 `.git`。解决方式是：**显式添加 `recursive_delete` 到 deny_operations，同时配合写前确认。**

3. **VFS 的快照超时设置太短，导致大文件写入失败**  
   在生成 200MB 测试日志文件时，Agent 的写入操作因为超时被 VFS 回滚，但 Agent 并未收到明确错误，导致它认为文件已生成，后续流程出现“文件不存在”的诡异报错。调整 `vfs.timeout` 到 60 秒，并开启工具调用错误回传后恢复正常。

## 可复用建议：面向不同场景的安全配置模板

- **纯读场景（数据分析、报告生成）**：只开启 `file_read`，`allowed_paths` 指向数据目录，禁用所有写、删、Shell 操作。零风险。
- **半自动本地开发辅助**：限制读写到项目根目录，开启 `write_gate` + `recursive_delete` 禁用，隐藏文件不可见，Shell 工具完全关闭。安全性远高于“允许所有，事后回滚”。
- **全自动 CI/脚本 Agent**：使用专用 Runner 账户，进一步通过 Linux 用户权限隔离（如 `chroot` 或 Docker 容器），OpenClaw 沙盒作为第二层防御。不要在 Agent 宿主机上保留重要未备份数据。

## 总结

OpenClaw 的沙盒安全模型不是一道“开关”，而是一组可组合的栅栏：路径白名单限制视野，操作黑名单阻断危险动作，VFS 层提供原子性与隔离，写前确认让人类保留最终裁判权。当你下一次犹豫“能不能让 Agent 碰我文件”时，不妨先问自己：**我配置的栅栏有几道？每道是否真正生效？** 最好的安全策略，是假设 Agent 每一次输出都包含恶意，但你的执行边界让它什么也破坏不了。

---

