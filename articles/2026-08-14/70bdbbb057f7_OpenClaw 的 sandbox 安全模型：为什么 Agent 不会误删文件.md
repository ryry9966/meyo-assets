---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 33054
source: 综合讨论
publishedAt: 2026-08-14
---

# OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件

## 背景

在 OpenClaw 这类 Agent 系统里，模型不仅会回答文本，还会通过 MCP/插件真正调用工具：读写文件、执行命令、操作项目目录。文件删除是其中风险最高的一类操作。一个幻觉路径、一段被注入的 prompt、一个插件 bug，都可能让 Agent 执行 `rm -rf` 或覆盖关键配置。

很多人第一反应是“在 prompt 里强调不要乱删”。但 prompt 约束不可验证、不可审计，模型也可能被上下文误导。工程上更可靠的做法，是在工具执行层做拦截。这篇文章聊一下 OpenClaw 的 sandbox 文件访问模型，以及为什么在默认配置下 Agent 不会“顺手”删掉你的文件。

## 问题

Agent 误删文件的来源通常有几种：

- 模型幻觉出错误路径，比如把 `/tmp/test` 写成 `~/test` 或 `/`。
- MCP server 暴露了过宽的文件系统权限。
- 插件绕过 OpenClaw 的受控工具，直接调用 `child_process` 或 shell。
- 自动化任务里路径变量拼接错误，导致删除范围越界。

如果只依赖 prompt 或“让模型小心”，这些问题依然会间歇性出现。真正有效的思路是：**默认拒绝，显式授权，删除走软删除，全程审计**。

## 做法：OpenClaw 的 sandbox 配置

OpenClaw 的 sandbox 不是简单的“禁用命令”，而是在文件访问层做了几件事：

1. **所有文件操作统一走工具代理**  
   默认不暴露原生 shell 给 Agent，只暴露 `file_read`、`file_write`、`file_delete` 等受控工具。这意味着 Agent 不能直接执行 `rm -rf`，必须调用 `file_delete`，而 `file_delete` 会先经过路径策略检查。

2. **默认只读，写操作白名单化**  
   典型配置如下：

   ```yaml
   # ~/.openclaw/sandbox.yaml
   mode: strict
   default_policy: read-only
   writable_paths:
     - ~/projects/current
     - /tmp/openclaw-sandbox
   deny_list:
     - ~/.ssh
     - ~/.aws
     - ~/.env
   destructive_ops:
     delete: ask
     overwrite: ask
     rename: allow
   trash:
     enabled: true
     trash_dir: ~/.local/share/openclaw/trash
   ```

   在这个配置下，Agent 可以读大部分路径，但写操作只允许 `~/projects/current` 和 `/tmp/openclaw-sandbox`。删除和覆盖默认是 `ask`，需要用户确认；重命名允许，但目标路径也必须在白名单内。`deny_list` 进一步把敏感目录排除，即使未来误配了更宽的白名单，也优先拒绝访问。

3. **软删除兜底**  
   即使某个删除操作被允许执行，OpenClaw 默认也会把它移到 `trash_dir`，而不是直接 `unlink`。这样即使策略判断失误，文件仍然可以恢复。

4. **路径规范化和符号链接校验**  
   OpenClaw 在匹配白名单前会做 `realpath` 解析，展开 `..`、`~` 和符号链接。举例，`~/projects/current` 里如果有个软链接指向 `~/.ssh`，Agent 尝试通过这个软链接写文件，会被识别为越界操作而拒绝。

5. **审计日志**  
   每次被拒绝的操作都会记录到审计日志，包括时间、工具名、路径、策略命中的规则。这样你可以事后排查 Agent 为什么尝试访问某个路径，而不只是看到“操作被拒绝”的模糊提示。

## 踩坑点

在实际使用中，有几个容易踩的坑：

- **白名单路径给得太宽**  
  如果你把 `writable_paths` 配成 `~` 或 `/`，等于没有 sandbox。白名单原则是最小化：只给当前任务需要写入的目录。

- **符号链接逃逸**  
  虽然 OpenClaw 会做 `realpath` 校验，但如果你的 `deny_list` 没有覆盖所有敏感目录，或者某个插件自己实现了文件操作，符号链接仍可能绕过检查。所以不要只依赖路径白名单，还要禁用绕过 OpenClaw 工具层的原生 shell 工具。

- **插件直接调用系统 API**  
  很多 MCP server 或插件为了省事，直接用了 `child_process.exec` 或 `os.remove`。这类调用不经过 OpenClaw 的 sandbox 代理，策略不会生效。建议只安装你信任的插件，并检查它们是否声明了“通过 OpenClaw tools 访问文件系统”。

- **`mv` 到临时目录后清理**  
  有些自动化流程会把文件先移到 `/tmp`，然后定期清理 `/tmp`。如果 OpenClaw 的软删除目录被同一套清理逻辑扫到，可能造成二次删除。建议把 `trash_dir` 放在独立目录，并排除在系统清理任务之外。

- **确认操作被自动化“自动确认”**  
  如果你在 `destructive_ops` 里把 `delete` 配成 `allow` 或 `auto_confirm`，sandbox 的保护会大幅削弱。除非你在跑完全受控的 CI 任务，否则保留 `ask` 是最稳妥的。

## 可复用建议

如果你要在自己的 OpenClaw 环境里落地这套安全模型，可以参考这五条：

1. **默认只读，写白名单最小化**  
   每个项目只加当前项目目录，不扩大范围。

2. **禁止直接 shell，保留受控文件工具**  
   只暴露 `file_read`、`file_write`、`file_delete`，禁用 `shell` 或 `exec` 工具。

3. **删除/覆盖保持 `ask`，开启软删除**  
   即使误判，也有恢复机会。

4. **定期跑 sandbox 逃逸测试**  
   构造符号链接、路径穿越、根路径写入等用例，确认都被拒绝。

5. **审查审计日志**  
   每周看一眼 `~/.openclaw/logs/audit.jsonl`，重点看被拒绝的 `delete` 和 `write` 操作，发现异常路径及时调整 `deny_list`。

## 总结

OpenClaw 的 sandbox 安全模型不是靠 prompt 约束 Agent，而是在文件访问层做了“默认拒绝 + 白名单 + 软删除 + 审计”的组合。Agent 不会误删文件，不是因为它“足够聪明”，而是因为它在执行删除前就被策略拦了下来。对于工程化的 Agent 实践来说，这种可验证、可回滚的机制，比任何口头约束都更值得依赖。

当然，sandbox 也不是银弹。如果你给 Agent 开了过宽的 shell 权限，或者装了绕过工具层的插件，保护就会失效。把 sandbox 当成最后一道防线，配合最小权限和插件审查，才能让 Agent 真正安全地跑在你的工作目录里。

---

