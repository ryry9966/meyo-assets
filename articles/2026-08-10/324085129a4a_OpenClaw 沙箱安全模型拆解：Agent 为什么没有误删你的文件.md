---
title: OpenClaw 沙箱安全模型拆解：Agent 为什么没有误删你的文件
feedId: 32370
source: 综合讨论
publishedAt: 2026-08-10
---

# OpenClaw 沙箱安全模型拆解：Agent 为什么没有误删你的文件

当 Agent 被赋予“读写本地文件”的能力时，大多数人的第一反应不是兴奋，而是警惕——万一它删了不该删的东西怎么办？OpenClaw 在设计插件执行模型时，没有把安全感完全寄托在 Prompt 约束上，而是在运行时环境中引入了一个显式的沙箱模型。这篇拆解不会谈“通过精心设计的 System Prompt 来限制行为”，而是从执行边界、权限声明、路径隔离这几个工程层，说明 Agent 为什么在真实操作中很难越界。

## 背景：Agent 的文件操作到底危险在哪里

先明确风险来源。以一个典型 MCP Filesystem Server 为例，它暴露 `read_file` / `write_file` / `list_directory` / `delete_file` 这些工具。Agent 在推理过程中会自主选择调用哪些工具、传什么参数。如果没有强约束，以下情况都可能发生：

- 用户问“帮我总结最近一周的日志”，Agent 可能遍历 `/var/log`，甚至误匹配到 `.bash_history`；
- 用户说“清理临时文件”，Agent 可能把 `~/Documents` 下的所有 `.tmp` 文件删光，而其中一些是你手工改后缀保存的草稿；
- 更严重的是，目录遍历或 `rm -rf` 式操作一旦被注入路径穿越，后果完全不可控。

要解决的工程问题是：**哪怕 Prompt 被幻觉、被误解，工具层的执行也必须被限制在安全边界内。**

## OpenClaw 的 sandbox 安全模型如何设计

OpenClaw 的插件系统（包括内置 MCP 兼容封装）在文件和系统访问上引入了三层明确的分隔，合在一起构成一个沙箱机制。

### 1. 受控文件系统根（Controlled FS Root）

插件在执行文件操作时，并不是直接对接宿主机绝对路径，而是由一个 **SandboxFS** 中间层做路径解析。开发者或用户在注册插件时，需要显式声明一个 `allowedDirectories` 列表（可以是一个或多个绝对路径），例如：

```json
{
  "name": "local-docs",
  "type": "filesystem",
  "config": {
    "allowedDirectories": ["/home/user/projects/my-docs"]
  }
}
```

此后，Agent 通过这个插件实例发出的任何文件操作请求，都会强制把传入路径与声明的目录做前缀匹配。不匹配的直接拒绝，不会进入真实文件系统。这意味着就算 Prompt 遭到“注入”，让 Agent 去读 `/etc/passwd`，沙箱层也会在路径校验阶段就把它拦下来，根本不产生系统调用。

### 2. 操作权限白名单（Capability Allowlist）

`allowedDirectories` 解决了“能碰哪些地方”，权限白名单解决“能做什么动作”。OpenClaw 的文件系统适配器不会给每个实例都开放全部 CRUD。配置文件里可以看到类似这样的声明：

```json
{
  "allowedOperations": ["read_file", "list_directory", "write_file"]
}
```

如果一个插件只声明了 `read_file` 和 `list_directory`，那么 Agent 尝试调用 `delete_file` 时，会在调用分发层直接返回 `Operation not permitted`。这个拒绝过程和 LLM 推理完全无关——它是运行在工具调用路由层的纯逻辑判断。很多用户担心的“误删文件”从根本上就不存在，因为 `delete_file` 能力根本没有被授予 Agent。

### 3. 运行时权限不能自我提升

这是容易忽略的一点。OpenClaw 的插件声明在所有连接初始化阶段就已经被锁定，沙箱配置不暴露给 Agent 自身，Agent 也不具备修改 `config` 的任何工具。这意味着不存在“通过说话就让沙箱扩大范围”的可能性。哪怕 Agent 生成了一段让 MCP Server 修改配置的请求，适配器也不会提供对应的接口。这种不可变性是工程上的硬保证，而非提示词层面的软约束。

## 一个实践版本：配置你的安全文件插件

下面是一个可以直接复用的最小安全配置，适合想让 Agent 处理某个项目 Markdown 文档的场景：

**Step 1**：在 OpenClaw 的插件配置中新增一个 filesystem 类型插件，不要使用默认的“全部可访问”配置（如果存在）。

```json
{
  "plugins": {
    "project-notes": {
      "type": "filesystem",
      "config": {
        "allowedDirectories": ["/home/me/projects/notes"],
        "allowedOperations": [
          "read_file",
          "list_directory",
          "write_file"
        ]
      }
    }
  }
}
```

**Step 2**：将 Agent 对话中所有文件相关的指令都指引到这个插件实例（通过插件名称 `project-notes`）。不要混用其他通用文件工具。

**Step 3**：如果你一定要允许删除，单独创建一个只读 + 删除分离的配置，仅允许在特定的 `trash` 目录下执行删除操作，并对该目录做定期备份。

```json
{
  "cleanup-helper": {
    "type": "filesystem",
    "config": {
      "allowedDirectories": ["/home/me/projects/notes/.trash"],
      "allowedOperations": ["read_file", "list_directory", "delete_file"]
    }
  }
}
```

这样 Agent 即便被要求“整理无用文件”，它的破坏范围也被物理隔离在一个可控的回收目录内。

## 常见踩坑点

1. **符号链接穿越**：仅做字符串前缀匹配可能被符号链接绕过。如果你的 `allowedDirectories` 下存在指向外部目录的 symlink，Agent 通过该 symlink 可以访问到沙箱外内容。OpenClaw 的 SandboxFS 默认在路径解析前对安全目录进行 `realpath` 解析，并对解析后的路径再次做前缀校验。但需要确保你使用的适配器版本已经启用该特性，并在部署时避免在安全目录内放置指向敏感位置的软链接。

2. **allowedDirectories 配置过宽**：有的人图省事直接把 `"/home/user"` 放进去，这相当于放弃了隔离。建议粒度控制到具体项目目录，不要贪方便。

3. **在同一个插件实例中混合安全要求不同的操作**：一个插件同时有 `write_file` 和 `delete_file`，且作用于重要目录，风险显著上升。宁可拆分为多个小权限插件，通过工具选择来控制暴露面。

## 可复用建议

- **拆分原则**：一个插件 = 一个目的 = 最小的目录 + 最少操作集合。互不信任的任务不要合并在同一实例。
- **定期审查 allowedDirectories**：随着项目演进，会积累一些旧目录，定期清理。
- **开启审计日志**：OpenClaw 可以记录实际触发的文件操作（尤其是写入和删除），方便追溯行为。
- **删除动作延迟策略**：如果业务允许，可以在 `delete_file` 的实现层增加一个“移动到回收站”的逻辑，沙箱内不做真实删除。很多错误得在事后恢复时才会被察觉到。

## 总结

OpenClaw 的沙箱模型并不是在 Prompt 里反复强调“不要乱删文件”，而是通过 **受控文件系统根 + 操作权限白名单 + 运行时不可变配置** 这三层机制，让 Agent 根本没有能力碰触超出声明的范围。安全感的建立不来自对 LLM 行为的猜测，而是来自限制执行面的确定性。如果你的 Agent 还在使用“全盘可读可写”的文件工具，不如花 20 分钟做一次权限收缩，这比任何防御性提示词都更可靠。

---

