---
title: 避免 Agent 误删文件：实现本地目录白名单的工程实践
feedId: 30891
source: 综合讨论
publishedAt: 2026-07-29
---

## 背景：当 Agent 拿到了文件系统的钥匙

越来越多的自动化 Agent 通过 MCP Server 或插件获得了文件系统操作能力——读配置文件、写日志、管理临时产物。这些工具让 Agent 变得“能干”，但也带来了最朴素的担忧：一个被幻觉误导的指令，或者一段失控的脚本，是否会把宿主机器上的关键文件删掉？

在没有护栏的情况下，Agent 发起的 `rm -rf /home/user` 或覆盖 `/etc/config` 并不会比普通进程多一层保护。我们通常会给人类操作加确认，但自动化流程不能依赖人工，必须在工具层就截断越权行为。

## 问题：工具暴露了，边界却没划

典型的 Agent 文件工具会接受任意路径作为参数，例如 `read_file("../../.env")` 或 `write_file("/etc/cron.d/evil", ...)`。如果仅靠提示词约束“不要碰系统文件”，那只是软限制，不可靠。真正的防线应该是一段精确的代码：允许访问的目录白名单。文件操作在执行前必须经过路径校验，凡是落在白名单之外的请求，一律拒绝。

这个需求看起来简单，但路径检查在实战中很容易踩坑：符号链接绕过、`..` 路径穿越、`~` 展开、Windows 盘符、大小写敏感差异……如果只是简单做 `startswith("/safe")`，那几乎形同虚设。

## 做法：实现一个可复用的文件访问守卫

假设我们在 OpenClaw 环境中为 Agent 暴露一个 MCP 文件服务。需要在这个服务的每一个文件操作入口处嵌入一层“守卫”，其核心逻辑如下：

1. **白名单配置**：用一组绝对路径表示允许访问的目录，例如 `["/data/agent_workspace", "/tmp/agent_cache"]`。
2. **路径规范化**：将用户传入的路径和白名单路径都转换成解析符号链接后的绝对路径，消除 `..` 和相对路径的干扰。
3. **归属判断**：确认规范化后的目标路径是白名单目录的子路径（或等于白名单目录自身）。

我们用一个精简的 Python 实现来承载这些原则：

```python
from pathlib import Path
from typing import List

class FileGuard:
    def __init__(self, allowed_dirs: List[str]):
        # 将白名单目录都解析为真实绝对路径，避免符号链接差异
        self.allowed_real = [Path(d).expanduser().resolve() for d in allowed_dirs]

    def is_allowed(self, target: str) -> bool:
        p = Path(target).expanduser().resolve()
        # 检查 p 是否为某个白名单目录的子路径，或恰好等于白名单目录
        return any(
            p == allowed or allowed in p.parents
            for allowed in self.allowed_real
        )
```

在 MCP 工具函数（如 `read_file`, `write_file`）中，我们直接调用：

```python
guard = FileGuard(allowed_dirs=["/data/agent_workspace"])
if not guard.is_allowed(file_path):
    raise PermissionError(f"路径越权: {file_path}")
```

这样做之后，即使 Agent 传入了 `"/data/agent_workspace/../../../etc/shadow"`，`resolve()` 会将其还原为 `/etc/shadow`，它不在 `/data/agent_workspace` 的子路径中，操作就会被阻止。同时，白名单目录本身的符号链接也会被提前解析，避免“真实路径不在白名单但软链接在白名单下”造成漏判或误判。

## 踩坑点：真实环境比示例脆弱得多

在工程实践中，仅仅有上面的代码还不够，有几个容易忽略的地方：

- **工作目录相对路径**：如果传入的是相对路径 `"config.yaml"`，`Path.resolve()` 会基于进程当前工作目录（CWD）解析。这时需要确保 CWD 固定为白名单内的某个目录，或在守卫中将相对路径主动拼接一个安全的基路径，否则其解析结果可能跑到其他目录去。
- **`~` 展开**：`Path.expanduser()` 仅对以 `~` 开头的路径有效，但如果白名单或输入中出现了用户家目录的变形（如 `$HOME`），需要额外处理环境变量，或统一在服务启动时预解析。
- **符号链接的两种视角**：如果一个白名单目录本身是符号链接，例如 `/data/safe -> /mnt/real_safe`，我们希望 Agent 只能访问 `/mnt/real_safe` 下的真实路径，这时把白名单 `resolve()` 为 `/mnt/real_safe` 是正确的。但如果你希望保留符号链接名作为白名单锚点，就要确保输入路径的 `resolve()` 也解析出同一真实路径，否则会出现“真实路径不匹配”的误杀。简言之，保持白名单与输入路径使用同一套解析规则。
- **多盘符系统（Windows）**：跨盘符时 `parent` 链不会自然延伸到根，`c:\data` 和 `d:\work` 互不相干。用 `pure_relative_to` 可能抛出异常，最好回归“绝对路径 + 共同前缀”检查，或借助 `os.path.commonpath` 辅助判断。
- **竞态条件**：检查时刻路径合法，检查后文件被替换为恶意软链接（TOCTOU）。对于高安全场景，可以在 `open()` 之后通过文件描述符再次确认真实路径，但大多数 Agent 工具场景可以选择先保证逻辑正确性，代价是稍微增加复杂度。

## 可复用建议：将护栏抽象为通用中间件

在 OpenClaw 这种多 Agent、多 MCP Server 的架构中，不应在每个工具里重复实现守卫逻辑。推荐的做法是：

- **环境变量注入**：通过 `AGENT_WHITELIST_DIRS` 环境变量传递白名单，部署时由平台写入，无需硬编码。
- **装饰器/中间件**：为 MCP Server 的文件操作工具定义统一的权限检查装饰器，例如 `@guard.require_allowed`，内部自动完成路径解析与阻断。
- **拒绝日志**：记录每一次越权尝试，包括请求路径、解析后的真实路径、触发时间，便于事后排查 Agent 是否产生了危险意图。
- **默认拒绝**：白名单可配置但必须启动时设定，不允许运行时动态缩小，避免被 Agent 篡改；默认情况若未配置白名单，则应拒绝所有文件访问，而不是放行。
- **与 OpenClaw 的整合**：OpenClaw 的 Agent 配置中可以将白名单作为资源策略的一部分，例如通过 `env` 字段传递给对应的 MCP Server 进程，实现一个沙箱化的工具集。

## 总结

给自动化脚本加本地目录白名单，本质是把“文件系统可访问范围”从开放全集收缩到一个精心维护的子集。这无关任何特定平台，而是一条通用工程原则：**永远不要完全信任 Agent 的决策，用强制性边界去兜底。** 实现这个边界只需几十行严谨的路径规范化代码，但它能让你在享受 Agent 文件操作便利的同时，不用半夜惊醒想起某个误删指令。

在 OpenClaw 架构下，将这种文件访问护栏内建为 MCP Server 的基础能力，成本极低，收益却是对安全底座的有效强化。下一次你给 Agent 增加 `read_file` 或 `write_file` 工具时，记得先把这道锁装上。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/d922b17ce503a7a0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/8f0e933f66a129b0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-29/a9278459d1325517.png)

