---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 35253
source: 综合讨论
publishedAt: 2026-08-29
---

## 背景

在 OpenClaw 里接 Agent、MCP 或自动化任务时，最让人不踏实的往往不是 Agent“不够聪明”，而是它在没有人类确认的情况下直接操作文件。尤其是删除动作：一句“清掉临时文件”，如果路径解析错误、通配符展开异常，或者某个插件把 `~` 当成了普通目录，就可能造成不可逆损失。

OpenClaw 的 sandbox 不是靠“模型自觉”来保证安全，而是在工具执行层强制加了一道边界。它的核心思路是：Agent 可以提出操作，但真正落到文件系统之前，必须经过路径约束、白名单和删除策略过滤。

## 问题

真正需要解决的不是“Agent 会不会变坏”，而是：

- 合理指令 + 异常路径，例如 `tmp/*` 被展开成隐藏目录；
- 插件或 MCP Server 拿到过宽的 shell 权限；
- 删除动作直接调用 `rm`，没有回收站或快照兜底；
- 路径穿越、符号链接逃逸，绕过配置中的只读目录。

这些场景里，Agent 并没有恶意，但结果可能和“误删”一样严重。

## 做法 / 步骤

### 1. 让 Agent 只拥有一个受控工作区

OpenClaw 的工具执行层默认把文件操作限定在 workspace 内。不要把所有文件操作都映射到 home 目录。一个可用的基线是：

```yaml
sandbox:
  workspace: ~/.openclaw/workspace
  writable:
    - ~/.openclaw/workspace
    - ~/.openclaw/tmp
  readonly: true
  deny_paths:
    - ~/.ssh
    - ~/.aws
    - ~/.config/openclaw
```

这样即使 Agent 被插件诱导，也只能写进两个可写目录，系统敏感目录直接拒绝。

### 2. 删除动作先走回收站，而不是直接 `rm`

把危险删除映射成“移动到回收站”。OpenClaw 的 sandbox 配置中可以把 `rm`、`unlink`、`rmdir` 强制重定向到 `.trash`：

```yaml
sandbox:
  allow_delete: false
  trash_dir: ~/.openclaw/.trash
  trash_keep_days: 7
```

这样做的好处是：误删不是“没了”，而是“移走了”。你可以定期清理 `.trash`，但 Agent 不会直接触发不可逆删除。

### 3. 高危操作加入审批门

批量删除、通配符、非 workspace 路径，不应该让 Agents 自动通过。可以设置审批策略：

```yaml
approval:
  required_for:
    - "rm"
    - "rmdir"
    - "unlink"
    - "glob:*.sh"
  auto_approve_timeout: 0
```

`auto_approve_timeout` 设为 `0`，表示没有超时自动放行。很多误删事故不是工具不行，而是审批超时后被当成“默认允许”。

### 4. 阻断路径穿越和符号链接

沙箱层必须做路径规范化。对于 `../`、绝对路径指向 workspace 外、符号链接指向敏感目录的情况，应直接拒绝。配置中通常有类似：

```yaml
sandbox:
  symlink_policy: deny
  allow_absolute_paths: false
```

这是非常基础但容易被忽略的一层。没有它，白名单形同虚设。

### 5. MCP / 插件按最小能力授权

MCP Server 会声明自己需要哪些工具。不要因为一个文件搜索插件要读目录，就给它 `shell` 或完整 `filesystem` 权限。对插件做一次能力清单审查：它能读哪些路径，能否写，能否执行删除，是否有 shell 逃逸可能。凡是不需要写的插件，只给读权限；凡是只需要当前项目目录的，不要映射整个 home。

## 踩坑点

### 1. 把整个 home 目录映射进 workspace

这等于放弃了沙箱的路径隔离。`~/` 下通常有配置文件、密钥、浏览器数据，任何越界写入都可能影响真实环境。

### 2. 容器里跑了沙箱，但挂载了宿主目录

容器隔离并不等于文件系统安全。OpenClaw 如果跑在容器里，又通过 volume 挂载了宿主机目录，那么容器内的只读或垃圾回收策略不会保护宿主机。要么不挂载，要么在宿主机做快照。

### 3. MCP 工具自带 shell，绕过 OpenClaw 的沙箱

有些插件会通过 `child_process` 或 `exec` 执行命令。即使 OpenClaw 限制了文件工具，插件内部的 shell 仍可能直接操作文件。要检查插件是否声明了 shell 能力，必要时禁用或降权。

### 4. 垃圾回收策略太激进

`trash_keep_days` 设得太短，或者自动化任务定期清理回收站，会让“暂存”变成“真删”。分离 Agent 的回收站和手工清理任务，避免误清。

### 5. 审批策略被无条件放行

测试环境为了方便，`auto_approve_timeout` 设置成 1 秒，结果生产环境没改回来。审批门必须保持同步配置，自动化脚本里不要写死允许。

## 可复用建议

- **分层设防**：路径白名单是第一层，回收站是第二层，审批流是第三层。不要只依赖一层。
- **文件操作统一入口**：让所有插件的删除动作都走 OpenClaw 的 sandbox 工具，而不是各插件自己实现。
- **写操作使用白名单，读操作可以只读挂载**：不要用黑名单去枚举“危险目录”，容易漏。
- **做一次故障演练**：故意让 Agent 删除 `~/.ssh` 或 `~/.aws`，检查是否被拦下。能拦下才叫模型有效。
- **MCP 清单成为上线流程一部分**：凡新增插件，必须标记是否具备 shell、文件写入、网络访问。没有声明就不接入。

## 总结

OpenClaw 的 sandbox 安全模型，本质上不是“阻止 Agent 删除文件”，而是把删除变成**可恢复、可审计、可拦截**的操作。它通过在工具执行层做路径约束、回收站和审批门，避免了把安全寄托在模型意图上。真正可靠的 Agent 自动化，不是让 Agent 更小心，而是让它即使出错，系统也能兜底。

这条边界对 MCP、插件和自动化用户来说都一样：只要写操作被默认限制在受控工作区，删除动作被强制走垃圾回收，那么“误删文件”就从不可逆事故变成一次可回溯的异常。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/638f0fd87e19ccdf.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/5f4a6f750111f92e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/2feca446669e8411.png)

