---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 36609
source: 综合讨论
publishedAt: 2026-09-08
---

## 背景

给 Agent 接上文件系统工具的那一刻，风险就从“模型答错一句话”升级成了“工作目录少了一个文件”。我们团队在把 OpenClaw 用于日常自动化（日志清理、构建产物管理、批量改配置）时，最先被问的问题不是“它能不能干活”，而是“它会不会手滑删了别的东西”。这篇帖拆一下 OpenClaw sandbox 的分层设计，以及为什么误删在我们的实践中基本被消除了。

## 问题

误删不是模型“故意”造成的，而是三个因素叠加：

1. 模型对路径的幻觉——拼错目录、把相对路径当绝对路径；
2. 工具调用链里缺乏统一的权限裁决点；
3. 破坏性操作没有闸门就直接落盘。

只在 system prompt 里写“请小心操作”，等于把安全边界建立在概率上。

## 做法：四层叠加的 sandbox

**1. 命名空间隔离。** Agent 的所有文件工具运行在一个挂载的 workspace root 内，路径解析被强制限制在这个子树里。绝对路径越界、`..` 穿越、符号链接指向 sandbox 外的目标，都在路径规范化阶段被拒绝。这层在文件工具和 shell 执行器上同时生效。

**2. 权限分级与默认拒绝。** 工具按 read / write / execute / destructive 四档声明权限，policy 默认只开读。写权限要在 profile 里显式授权；destructive 类操作（删除、覆盖、批量重命名）默认走确认门，由用户 approve，或配置为“仅允许匹配特定 glob 的路径”。

**3. shell 命令拦截。** 模型绕过专用工具、直接调 shell 执行 `rm -rf` 是最常见的漏点。OpenClaw 在 shell 执行前做静态命令解析：命中删除/递归覆盖模式的命令，要么被改写为 sandbox 内的受控删除 API，要么直接拒绝并提示模型改用工具。

**4. 审计与回滚。** 所有写操作落盘前先记操作日志；破坏性操作把目标文件移入 sandbox 内的 `.trash`，而不是直接 unlink。出问题时按操作序号回滚即可。

配置上三步：profile 里指定 workspace root → 打开 write 权限并配置 destructive 确认策略 → 给 shell 拦截器加白名单（比如允许 `rm` 只作用于 `/tmp` 下的临时产物）。

## 踩坑点

- **root 挂太宽**：直接挂 `$HOME`，隔离层形同虚设。按项目挂子目录，宁可多建几个 workspace。
- **符号链接是头号逃逸路径**：早期我们没开 symlink 出界检查，agent 通过仓库里一个指向外部的软链写穿了目录。现在默认拒绝解析出界的 symlink。
- **MCP 侧门**：第三方 MCP server 自带的文件工具不走 OpenClaw 的路径裁决。接入前先审工具清单，或用代理层把它的文件访问包进同一套检查。
- **拦截是启发式的**：`find ... -exec rm` 这类管道可能绕过模式匹配。别把 shell 拦截当唯一防线，它只是四层中的一层。

## 可复用建议

把 Agent 当成“不可信新同事用的系统账号”来做权限设计：最小授权、默认拒绝、破坏性操作必须有闸门和回滚。另外，上 sandbox 之前先把工作目录纳入 git 或快照——这层兜底的价值比任何拦截器都高。

## 总结

OpenClaw 不依赖“模型表现良好”，而是假设它一定会犯路径错误、一定会尝试越权，然后用命名空间隔离、权限分级、shell 拦截、审计回滚四层把错误代价压到可恢复。实践里最大的体会是：sandbox 不是单个开关，而是一条从路径解析到落盘的完整链路——任何一环接了第三方工具或挂宽了目录，防线就会从那里开洞。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/603b11359f9e84ff.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/1cfb99ef9562229b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/222885098265c222.png)

