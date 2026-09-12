---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 37193
source: 综合讨论
publishedAt: 2026-09-12
---

## 背景

给 Agent 接上 shell 和文件工具之后，最怕的不是任务跑不通，而是跑"过"了。模型一次路径幻觉、一个错误的相对路径展开，就可能把工作目录之外的文件清掉。社区内测群里至少见过三起"agent 把构建产物连同源码一起删了"的复盘。OpenClaw 的设计前提因此很明确：**假设模型会犯错，执行层默认不信任模型输出。**

## 问题拆解

Agent 误删文件通常不是"想删"，而是三层失效叠加：

1. 进程边界失效：agent 进程能看到整个文件系统；
2. 路径解析失效：符号链接、`..`、相对路径把目标带出了预期范围；
3. 操作分级失效：删文件和读文件走同一条通道，没有额外的确认成本。

## OpenClaw 的做法

sandbox 模型由四层组成，从外到内：

**1. 文件系统隔离。** `openclaw sandbox init --workspace ./proj` 把 agent 进程关进容器：根文件系统只读，只有 workspace 以读写方式挂载进来。进程里看到的 `/` 不是宿主机的 `/`。

**2. 路径规范化强制拦截。** 所有 fs.write / fs.delete 调用在工具层先做 realpath 解析，最终落点（含符号链接的真实指向）不在 workspace allowlist 内就直接拒绝。这一层专门拦 symlink 逃逸——workspace 里被塞一个指向 `/etc` 的软链也没用。

**3. 操作分级 + 确认门。** 读操作免审；写操作限制在 workspace 内；删除/重命名/移动必须走 confirm gate，且只能作用于会话开始时登记过的文件。含 `rm -rf`、递归删除的命令默认整条拒绝，不做拆解执行。

**4. 会话级快照兜底。** 每个会话启动时对 workspace 打一次 git 快照。极端情况下前面三层全部失效，`openclaw workspace restore` 也能还原目录。

### 验证方法

不要相信配置，要亲手测：

```bash
# 应被拒绝：目标在 workspace 外
openclaw exec "rm -rf /tmp/demo"

# 应被拒绝：未登记文件
openclaw exec "rm ./proj/README.md"

# 应成功：登记过的临时文件
openclaw exec "rm ./proj/.tmp/cache.bin"
```

通过后到 audit log 核对每次拒绝的 reason 字段，确认是哪一层拦的。

## 踩坑点

- **把宿主机大目录挂进 workspace**：有人图省事把整个 home 挂进去，sandbox 形同虚设。挂载粒度务必最小化。
- **第三方 MCP server 绕过工具层**：MCP server 自带文件访问能力时不经过 sandbox 工具层，要在 MCP 配置里单独做路径白名单。
- **root 跑 daemon**：容器内提权后读写限制全部作废。用普通用户启动。
- **构建工具写 /tmp 被拦**：allowlist 收太紧会导致构建失败，正确做法是把 tmp/cache 目录显式加入可写白名单，而不是整体放宽。

## 可复用建议

- default-deny，最小可写集，按需放开；
- 破坏性操作永远走 gate + 事前登记，没有例外；
- 会话级快照成本极低、收益极高，无脑开；
- 把 sandbox 规则当代码维护：进版本库、过 review，并配一组"越狱测试"用例跑进 CI；
- 新接入的 MCP 工具，先在沙箱里跑一轮恶意用例回归再上线。

## 总结

Agent 不误删文件，从来不是靠模型更聪明，而是靠执行层把"模型会犯错"当成第一假设。四层防护里，快照兜底最便宜，路径规范化拦截性价比最高，任何部署都建议默认开启。安全配置的验收标准只有一条：你亲手试过让它删不该删的东西，并且它拒绝了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/bf06d2362501486f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/17a4c8c4a40c227b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/7c8269f461865c81.png)

