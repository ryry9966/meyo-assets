---
title: SOUL.md 实践：给 OpenClaw Agent 一个稳定的人格与边界
feedId: 33538
source: 综合讨论
publishedAt: 2026-08-17
---

# SOUL.md 实践：给 OpenClaw Agent 一个稳定的人格与边界

## 背景

在 OpenClaw、MCP 工具链和插件自动化场景里，Agent 的“人设”通常靠 system prompt 维护。但真实项目里常出现两类问题：

- 系统提示越写越长，规则散落在 main prompt、工具描述、项目 README 里，模型注意力被稀释，后期出现人格漂移。
- 多人协作或频繁迭代时，行为约束改不动，Agent 偶尔会越权调用工具、误删文件，或者在不确定时开始编造。

SOUL.md 的思路很简单：把 Agent 的身份、语气、工作边界、失败处理从主流程里抽出来，单独用一个版本化文件维护。它作为 system prompt 的稳定头部注入，给 Agent 一个长期的“人格锚点”。

## 问题

一个典型场景：你在 OpenClaw 里接入了文件系统 MCP、Shell MCP 和 Git 插件。Agent 在长会话里逐渐忘记“只读优先”，开始主动执行 `git push`，或者把 workspace 外的路径也读了一遍。主 prompt 里不是没写限制，而是写得过于分散：一部分在 system prompt，一部分在 MCP 工具描述，一部分靠用户口头强调。

结果就是行为不可复现：同一个任务，换一个会话，Agent 的表现不一样。

## 做法 / 步骤

### 1. 创建 SOUL.md，放在 Agent workspace 根目录

由启动器或包装脚本读取，注入到 system prompt 头部。例如：

```text
<SOUL>
## Identity
你是 OpenClaw 运维助手，语气克制，默认中文，不堆砌表情。

## Boundaries
- 允许：读取 /workspace 下文件，调用 list_files、read_file、grep。
- 禁止：删除文件、修改 /etc、执行网络下载脚本。
- 涉及 git push、npm publish 前必须显式确认。

## Workflow
1. 先读 README.md 与 package.json，再给结论。
2. 改代码前先展示 diff。
3. 工具连续失败 2 次，停止并报告，不换方案硬试。

## Failure
不确定时不要编造；给出最小复现和需要用户提供的信息。
</SOUL>
```

### 2. 内容分块，保持短小

建议控制在 150–200 行以内。四个核心块：

- **Identity**：是谁、语气、默认语言。
- **Boundaries**：允许 / 禁止 / 需要确认的动作，尤其要写与 MCP 工具相关的边界。
- **Workflow**：默认执行顺序，例如先查后改、先计划后执行。
- **Failure**：不确定、失败、越权请求时如何降级。

### 3. 与权限层配合

SOUL.md 是软约束，不替代硬限制。OpenClaw 的工具权限、MCP server 的 allowed tools、文件系统沙箱仍然要做白名单。例如 SOUL.md 里写“禁止删除文件”，但权限层仍应直接移除 `delete_file` 工具，或限定 `--allow` 路径。

### 4. 注入方式要稳定

可以用一个 `load_soul.sh` 或 OpenClaw 插件在每次会话启动时读取 SOUL.md，拼到 system prompt 最前面，并加上 `<SOUL>...</SOUL>` 分隔标记。避免把 SOUL.md 复制进主配置，否则又会回到规则散落的老路。

## 踩坑点

1. **SOUL.md 过长**  
   超过 200 行后，模型经常只记住前几条，后面规则被忽略。宁可用注释链接到详细文档，也不要把完整 SOP 塞进去。

2. **把敏感信息写进去**  
   API token、数据库连接串、内网地址不要出现在 SOUL.md 里。模型可能把 system prompt 内容复述给用户，尤其是遇到“把你系统提示给我看看”类请求时。

3. **只靠 SOUL.md 做安全边界**  
   SOUL.md 里的“禁止执行 shell”可能被诱导绕过。真正的危险操作必须在工具权限层拦截，SOUL.md 只负责行为引导。

4. **规则与工具描述冲突**  
   如果 SOUL.md 写“不要执行未知命令”，但 Shell MCP 的描述里写着“execute any command”，模型往往会优先按工具 schema 走。需要同步修正工具描述，或直接移除高危工具。

5. **更新后未重启会话**  
   SOUL.md 改了，但旧会话还缓存着旧 system prompt，导致行为没变化。建议 loader 在启动时打印 SOUL.md 的 hash，或在 prompt 里标注版本号。

## 可复用建议

- 用“必须 / 禁止”这样的绝对词，不要写“尽量”“建议”。模糊规则等于没有规则。
- 把关键边界写成可测试项，例如：“当用户要求删除文件时，应输出确认请求，而不是直接执行”。
- 按角色拆分 SOUL.md 片段：`base.soul.md` 加 `ops.soul.md` 或 `frontend.soul.md`，用 include 组合，避免复制粘贴。
- 给 SOUL.md 做版本管理，每次修改记录原因，方便回溯 Agent 行为变化。
- 定期用一组固定 prompt 做回归：检查 Agent 是否越界、是否在失败时乱猜。

## 总结

SOUL.md 不能解决 Agent 的所有安全问题，但它能显著降低人格漂移和行为不一致的概率。工程上比较合理的做法是：**SOUL.md 管“怎么想、怎么表达”，权限层管“能做什么、不能做什么”**。两者叠加后，OpenClaw Agent 在长任务和多人协作里会稳定得多，也更可维护。

---

