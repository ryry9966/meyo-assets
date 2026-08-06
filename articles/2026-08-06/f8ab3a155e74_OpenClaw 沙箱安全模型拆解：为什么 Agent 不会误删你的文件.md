---
title: OpenClaw 沙箱安全模型拆解：为什么 Agent 不会误删你的文件
feedId: 31900
source: 综合讨论
publishedAt: 2026-08-06
---

## 背景：当 Agent 拿到文件系统的钥匙

在 MCP 工具调用链里，一旦 Agent 通过 shell、文件读写插件拿到主机文件系统的访问权，一个很现实的问题就摆到台面上：**如何保证它不会因为 prompt 注入、上下文误解或工具描述偏差，执行类似 `rm -rf /` 的操作？**

社区里常见的做法是“信任模型”——要么全靠用户审慎授权，要么给 Agent 一个独立容器随便折腾。但这两种方案在本地自动化工作流里都不够工程化：前者打断感太强，后者太重，且没法直接访问宿主机上需要处理的真实目录。

OpenClaw 的 sandbox 安全模型在设计时走了一条中间路线：**给 Agent 一个“看起来像真实文件系统，但实际上每一层操作都被拦截、审计并可回滚”的执行环境。** 这篇文章就把它的实现方式、配置步骤和踩过的坑整理出来。

## 问题：误删不只是 prompt 注入的锅

很多人以为危险操作只来自恶意 prompt，但在实际跑 Agent 任务时，我们遇到过更隐蔽的场景：

1. **工具描述歧义**：给了一个 `clean_temp_files` 工具，内部实现是 `rm -rf $TEMP_DIR/*`，但 Agent 某次推理时误把 `TEMP_DIR` 理解成当前工作目录。
2. **上下文污染**：上一轮对话中用户随口说了一句“把所有旧版本都删掉”，Agent 在后续无关任务里复用了这个指令。
3. **路径拼接错误**：Agent 拼出了 `~/valuable_data/../*.tmp`，结果 shell 解析后删掉了不少正常文件。

这些都不是恶意攻击，而是当前 LLM 驱动的 Agent 在工程落地时必然会出现的“理解漂移”。所以安全模型不能只靠 prompt 围栏，必须在执行层做硬约束。

## 做法：三层沙箱，拦截在最后一道防线

OpenClaw 的 sandbox 默认关闭，但一旦开启，会对所有被标记为 `filesystem` 类别的 MCP 工具生效。它由三个层次组成：

### 1. 文件系统层隔离（OverlayFS + Copy-on-Write）
Agent 看到的是一个叠加层视图：底层是宿主机上你授权它读写的真实路径（只读挂载），上层是一个私有可写层。所有修改操作（创建、删除、重命名）默认只发生在上层，不会触碰底层真实文件。

这个设计的好处是：即使 Agent 执行了 `rm -rf /important`，它删的是叠加层里的“影子节点”，真实数据毫发无伤。任务完成后，你可以选择把上层的内容合并回去（commit），也可以直接丢弃。

### 2. 操作审计与预演（Dry-Run Mode）
任何落入 sandbox 管理的文件操作都会先进入一个预演阶段：Agent 的 tool call 会被拦截，生成一份操作清单（哪些文件会被创建、修改、删除），然后根据配置的策略决定下一步。默认策略是 `audit-only`，仅记录日志；正式环境我们建议设为 `simulate-first`，即第一次执行总是 dry-run，Agent 看到结果后二次确认才真正操作。

### 3. 用户确认门控（Human-in-the-Loop）
对于高危操作（删除非临时文件、修改隐藏文件、跨目录移动），可以配置必须弹出一个 CLI 确认提示或 HTTP callback。这个门控不是简单的 yes/no，而是会展示 sandbox 预演阶段生成的具体文件列表，让用户做最终裁决。

## 实践步骤：从配置到跑通一个安全任务

以下以一个本地文件整理 Agent 为例说明配置步骤。

**Step 1：启用 sandbox 并指定挂载点**
在 OpenClaw 配置文件 `agent.yaml` 里：

```yaml
sandbox:
  enabled: true
  root: /home/user/projects   # 允许 Agent 操作的根路径
  mode: overlay               # overlay | direct (禁用写隔离)
  writable_paths:
    - /home/user/projects/output
  audit: true
```

这里只把 `output` 目录挂为可写（在叠加层里），其余项目文件只读。

**Step 2：给文件操作工具打标签**
用 MCP 插件开发时，在工具定义里声明 `sandbox: filesystem`：

```json
{
  "name": "delete_old_backups",
  "description": "Remove backup files older than 30 days",
  "sandbox": "filesystem",
  "parameters": { … }
}
```

不声明该标签的工具不受 sandbox 约束，仍然可以直接操作宿主机文件——所以不要让未审计的脚本拿到写权限。

**Step 3：运行任务并观察审计日志**
启动 Agent 后，可以在 `~/.openclaw/sandbox/audit.log` 里看到类似：

```
[2025-01-12 10:23:45] DELETE /home/user/projects/archive/old.tar.gz (simulated)
[2025-01-12 10:23:50] User confirmed: yes → executed in overlay
```

原始文件 `old.tar.gz` 在底层并未被删除，只在叠加层标记为“已删除”。任务结束，如果确认无误会，执行 `openclaw sandbox commit` 将变更写回真实文件系统。

## 踩坑点：路径映射与持久化写入

1. **Agent 看到的路径≠真实路径**：叠加层会使某些临时文件在 sandbox 里可见，但在宿主机上找不到。Agent 如果在任务中途用 `pwd` 或读取 `/proc/self/root` 推路径，可能会出错。建议在工具 prompt 里明确告知工作目录的标准化路径。
2. **跨设备写入问题**：如果 `writable_paths` 和 `root` 不在同一个文件系统上，overlay 模式需要额外配置 `workdir`，否则挂载会失败。推荐把工作目录统一放在一个 ext4/xfs 分区下。
3. **不要 commit 太早**：一些用户嫌麻烦，每次任务完直接 `commit`，结果把 Agent 误删后的“无此文件”状态也写回了真实文件系统。建议始终先用 `openclaw sandbox diff` 查看叠加层变更清单，确认无误后再合并。

## 可复用建议

- **最小化只写权限**：即便有 sandbox 保护，也只给 Agent 开放必需的 `writable_paths`，不要图省事把整个家目录挂进去。
- **分层管理临时产物**：让 Agent 把生成的文件输出到 `output/` 或 `staging/`，和源文件物理隔离，便于事后清理或回滚。
- **配合版本控制**：在执行可能大规模改动的任务前，先 `git stash` 或打 tag，叠加层保护的是文件系统，不是 Git 历史。
- **日志告警**：把 `audit.log` 接入你的异常检测管道，一旦出现大量连续 `DELETE` 操作就触发通知，不必等到事后复盘。

## 总结

OpenClaw 的 sandbox 安全模型并没有发明新的隔离理论，但它把 **OverlayFS、操作审计、人机协同确认** 这三块砖，以很低的配置成本砌进了 Agent 工具调用的执行路径里。对于日常的文件整理、批量重命名、日志清理等自动化任务，它能让你敢于把真实数据交给 Agent 编排，而不是只能小心翼翼地做个“本地演示”。

在 Agent 走向更复杂的自主操作之前，这种可审计、可随时叫停的执行环境，是一道值得现在就去设置的护栏。

---

