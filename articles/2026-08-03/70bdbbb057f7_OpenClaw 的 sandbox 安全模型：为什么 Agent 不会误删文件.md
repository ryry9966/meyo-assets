---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 31467
source: 综合讨论
publishedAt: 2026-08-03
---

## 背景

在 Agent 自动化工具链里，“AI 把文件删了”一直是个经典恐怖故事。很多用户的真实痛点不是 Agent 能力不够，而是**不敢放开权限**。尤其在 MCP、多工具并行调用、批量脚本自动执行时，一个 `rm -rf` 级别的误操作，足以让整个自动化方案被同事和老板拉黑。

OpenClaw 的做法不是让 Agent “小心一点”，而是从工具调用层建立了一个沙箱边界。这篇文章从工作目录、工具权限、以及容器边界三个层面拆解它，并给出实际可复用的验证步骤。

## 问题

先说清楚问题。在传统 Agent 设计里，LLM 拿到 shell 权限后可以读写任意路径。为了完成“修改配置”这类任务，工具集往往开放 `run_command` 或 `write_file`。一旦指令被注入，或者模型上下文长了之后出现幻觉，误删就发生了。

OpenClaw 把这个问题拆成了两层来解决：

1. 大部分文件操作只允许在 workspace 沙箱内进行；
2. 需要脱离沙箱的操作显式授权。

## 做法：理解沙箱边界

OpenClaw 默认将当前工作目录（workspace）视为沙箱根。要确认这一点，可以直接看环境变量：

```bash
echo $OPENCLAW_WORKSPACE_DIR
# 例如 /home/user/agent-workspace
```

在代码层，OpenClaw 将工具分为三类：

- **只读工具**（`read_file`、`list_dir`）：可在全路径内执行，但无写入能力；
- **写工具**（`write_file`、`edit`）：强制解析路径到 workspace 内，再执行对应操作；
- **逃逸工具**（`execute_v2` 等 shell 执行类）：需要 **confirm:true** 显式授权，或配置了允许列表。

真正核心的拦截逻辑是一个内核级的路径解析器。所有文件写入操作都会先经过它，将相对路径转换为绝对路径后检查前缀是否为 workspace，如果不是，直接返回 `EPERM` 错误码。此外还处理了 `..` 路径穿越、符号链接逃逸，保证 `cd /tmp` 后 `write_file` 也不会写到沙箱外面。

## 步骤：实机验证

在一个干净的 Docker 容器中验证一下：

```bash
docker run --rm -it -v $(pwd)/workspace:/workspace openclaw/openclaw
```

进入 shell 后：

```bash
# 在 workspace 内读文件，OK
cat /workspace/test.txt
openclaw: read_file /workspace/test.txt

# 试图写 workspace 外的文件
openclaw: write_file /etc/passwd "hacked"
# 返回：Error: EPERM: write blocked by sandbox policy
```

再验证 shell 工具：

```bash
openclaw: run_command "rm -rf /workspace/app"
# 若未指定 confirm，返回：Request denied. Confirm required.
```

## 踩坑点

我实际跑了一轮，踩到了几个比较隐蔽的坑：

### 1. 配置了 `workspace.write-block` 但没重启服务
OpenClaw 对 YAML 配置的热更新并不完全支持，改 `openclaw.yaml` 后需要重启 daemon。

### 2. `::` 语法 vs `@` 语法的权限差异
OpenClaw MCP 配置里，工具引用有旧版 `::` 和新版 `@` 两种写法。旧语法不会带出沙箱上下文，有时会产生隔离失效的真实风险。排查时优先检查 mcp 配置是否全部使用新语法。

### 3. 路径解析是 API 层做的，不是 hook 层
也就是说，如果用 bind mount 把外部目录直接挂载到 workspace 内，沙箱仍然是对工作目录的路径前缀做判断，会认为挂载进来的外部文件属于 workspace 范围，写入是放行的。这个行为在文档中只被顺带提了一句，需要自己留意。

## 可复用建议

1. 每个服务一个 workspace，而不是所有 Agent 共享根目录；
2. 给高危工具配置 `confirm: true` 和 allowlist 两个条件同时满足才执行`;
3. 配置好错误告警：`EPERM` 或 `SandboxBlocked` 出现时立刻推送消息，这通常说明指令注入或路径异常；
4. 只给 MCP 暴露最小必要权限集，不在 config 里一次性 `allow_all`；
5. 定期抽查日志中 `sandbox_deny` 条目，了解 Agent 触发了哪些被拦截的操作。

## 总结

OpenClaw 的沙箱不是一个虚拟化“牢笼”，而是一个工程化的工具权限分界。它通过强制路径解析 + 工具级权限声明，让 LLM 的能力被约束在可审计、可回滚的范围内。最值得借鉴的点在于：它没有试图“教育模型别乱来”，而是让模型根本拿不到那扇门外的钥匙。

对团队来说，这套机制的意义在于：自动化 Agent 可以跑在生产环境旁边，而不用提前写好免责声明。

---

