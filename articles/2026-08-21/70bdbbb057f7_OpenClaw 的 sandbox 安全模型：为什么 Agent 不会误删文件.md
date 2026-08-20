---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 33947
source: 综合讨论
publishedAt: 2026-08-21
---

## 背景

在 OpenClaw 里跑 Agent 做自动化时，最常被问到的问题是：如果 Agent 执行了 `rm -rf /` 或者不小心删了项目文件怎么办？直接给 Agent 一个宿主机 shell 的风险确实很高，但 OpenClaw 默认并不会把宿主文件系统暴露给 Agent。它的安全思路不是“让 Agent 变聪明”，而是把 Agent 的操作圈进一个可回滚、受限的沙箱视图里。

## 问题：误删文件的风险点在哪里

Agent 误删文件通常不是因为它“想删”，而是这几个工程风险被放大：

1. **路径穿越**：模型生成 `../../etc/passwd` 或 `~/` 这类路径，直接访问宿主目录。
2. **命令拼接**：工具调用时参数未做严格校验，出现 `rm -rf $DIR/` 且 `$DIR` 为空的情况。
3. **符号链接逃逸**：沙箱内创建指向外部文件的软链，再通过写操作影响宿主文件。
4. **插件/ MCP 越权**：某些文件系统工具默认要求全局读写权限，一旦接入就等于裸奔。

OpenClaw 的 sandbox 主要解决前三点，第四点则需要靠插件权限策略来补。

## 做法/步骤

OpenClaw 的 sandbox 模型可以拆成四层，默认开启：

### 1. 只读根文件系统

沙箱启动时，`/etc`、`/usr`、`/bin`、`/lib` 等系统目录以只读方式挂载。Agent 即使执行 `rm -rf /usr`，也会得到 `read-only file system` 错误，不会影响宿主。

### 2. 工作区白名单 + Copy-on-Write

只有显式声明的工作区目录可写。OpenClaw 使用类似 overlayfs 的写时复制机制，Agent 的写操作落在 upper layer，不会直接修改 lower layer 的原始文件。这样即使删了工作区文件，宿主侧仍保留原始数据，可以随时回滚。

一个常见的部署配置示例：

```yaml
sandbox:
  backend: overlayfs
  root_readonly: true
  workspace: /var/lib/openclaw/sandbox/workspace
  allowed_writes:
    - /data/projects/demo
  forbidden:
    - /etc
    - /home
    - /root
```

### 3. 路径规范化与写操作代理

OpenClaw 在工具调用前会对路径做规范化处理：解析绝对路径、拒绝 `..` 穿越、检查符号链接真实目标。对于 `rm`、`mv`、`shred` 等危险命令，会改写为“移动到回收区”的 rename 操作，而不是直接 unlink。这样误删后还有恢复窗口。

### 4. 审计日志

所有文件操作都会记录到审计日志，包括操作类型、源路径、目标路径、是否被拦截。可以通过 `openclaw sandbox status` 查看挂载点和拦截统计；也可以直接进入沙箱执行 `mount` 或 `findmnt` 验证只读根是否生效。

验证步骤：

- 启动沙箱后，先跑一个无害的 `touch /etc/test` 确认被拒绝；
- 在 workspace 里创建测试文件，执行删除并观察回收区是否出现；
- 检查审计日志中是否有 `DENY` 记录；
- 创建符号链接指向 `/etc/hosts`，尝试写入，确认被路径规范化拦截。

## 踩坑点

1. **白名单太宽**：把 `/home` 整目录挂进沙箱，等于没隔离。只授权任务当前需要的子目录。
2. **符号链接没封住**：只限制路径字符串不够，还要解析软链真实目标，否则 `workspace/link` 可以指向外部文件。
3. **插件的全局读写权限**：有些 MCP filesystem 工具默认申请 `root` 下的读写，需要单独降级为只读 token 或代理接口。
4. **回滚快照没做**：只靠 overlayfs upper layer，时间一长 upper 被合并，原始文件也可能被覆盖。建议定期对 workspace 做快照。
5. **把沙箱路径当宿主路径**：Agent 说“删除了 `/tmp/xxx`”，可能是沙箱内的 `/tmp`，排查时别混淆。

## 可复用建议

- **最小权限原则**：默认只给 Agent 任务必需的目录，其他一律 deny。
- **写时复制 + 快照**：每隔固定时间对 workspace 做快照，保留 3~5 个版本。
- **危险命令统一走 trash**：`rm -rf` 改为移动回收区，设置 7~14 天保留期。
- **MCP 文件工具只读化**：如果任务只是读取，就用只读挂载；确需写入，再单独开一个受限目录。
- **审计告警**：关注 `rm -rf /`、`find -delete`、`shred`、大范围递归删除等模式，接入通知。
- **先在 CI 环境跑破坏性任务**：验证 Agent 的行为不会越过 sandbox 边界，再放到本地工作站。

## 总结

OpenClaw 的 sandbox 模型本质上是在回答一个工程问题：当自动化 Agent 拥有文件操作能力时，如何把“误删”从事故降级成可恢复的异常。它靠只读根、工作区白名单、写时复制和路径代理这四件事，把 Agent 的误操作限制在一个临时视图里。沙箱不是保险箱，但可以把爆炸半径缩到最小。真正安全的 Agent 实践，永远是默认最小权限 + 可回滚 + 可审计，而不是期待模型永远不犯错。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/8a1a4d606ef85515.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/0babbd388070079b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/5c50923a4a644436.png)

