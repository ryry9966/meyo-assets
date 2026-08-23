---
title: AI 助手的 USER.md：让 Agent 真正了解你是谁
feedId: 34406
source: 综合讨论
publishedAt: 2026-08-24
---

# AI 助手的 USER.md：让 Agent 真正了解你是谁

## 背景：Agent 为什么总是“不懂你”

在 OpenClaw 这类可编排 Agent、MCP 与插件的项目里，模型能力通常不是瓶颈，真正的短板是：它缺少关于“你”的稳定上下文。

很多配置只解决了“系统能做什么”，没有解决“为谁做、按什么规则做”。例如：

- 你习惯用 pnpm 而不是 npm；
- 生产环境禁止访问外网；
- commit message 必须是 `type(scope): subject`；
- 服务部署在 `192.168.x.x`，不能假设是本地回环。

这些信息如果每次对话都手动重复，会非常低效；如果全部塞进全局 system prompt，又会把上下文窗口撑爆。  
USER.md 就是为了解决这件事：给 Agent 一份关于你的、结构化、可检索、可维护的用户档案。

## 问题：上下文注入的碎片化

没有 USER.md 时，常见做法是：

1. 每次新会话重复说明偏好；
2. 把规则散落在聊天记录、插件配置、脚本注释里；
3. 让 Agent 自己猜。

这带来的问题很明确：规则不可复用；敏感信息与工作偏好混在一起；多设备、多角色场景下互相污染；Agent 无法主动获取关键约束，只能被动等待纠正。

## 做法：把 USER.md 当成“稳定事实层”

USER.md 不是系统提示词，也不是 todo list。它更适合记录三类内容：

### 1. 稳定事实
- 操作系统、Shell、包管理器、编辑器
- 目录结构习惯
- 网络/硬件限制
- 工作语言、时区

### 2. 决策偏好
- 技术选型倾向，例如“优先 SQLite，不引入独立数据库”
- 输出风格，例如“注释用中文，日志用英文”
- 执行风格，例如“删除/覆盖操作必须先确认”

### 3. 禁忌与边界
- 不允许访问的目录
- 不允许执行的命令
- 不允许上传或发送的内容

一个最小化 USER.md 示例：

```markdown
---
role: backend-developer
os: linux
shell: bash
package_manager: pnpm
timezone: Asia/Shanghai
---

# User Profile

## Environment
- dev machine: 192.168.1.20
- production: 10.0.0.0/24, no outbound internet
- editor: vscode

## Preferences
- Use pnpm, never npm/yarn.
- Commit message: `type(scope): subject`
- Code comments in Chinese, logs in English.

## Constraints
- Never delete files without explicit confirmation.
- Never read ~/.ssh or ~/.aws.
- For network requests, prefer 127.0.0.1 proxy if available.
```

## 让 Agent 真正“读到”它

文件写好后，关键是让 Agent 在合适时机加载，而不是全文注入每一轮对话。

建议配置三个触发点：

- **会话启动时**：注入 USER.md 的内容摘要；
- **任务开始前**：让 Agent 先读取 USER.md，并复述关键约束；
- **MCP 工具调用前**：在上下文中引用“偏好与禁忌”章节。

示例命令片段：

```yaml
# 示例：load-user-context 命令
steps:
  - read_file: "~/.config/openclaw/USER.md"
  - extract: "Constraints"
  - confirm: "确认已读取上述约束，并会在任务中遵守"
```

如果使用 MCP，也可以接入 filesystem MCP，让 Agent 显式读取 USER.md，这样权限边界更清晰，也方便按需加载。

## 踩坑点

1. **把 USER.md 当垃圾堆**  
   什么都往里写，最后变成几百行笔记。Agent 检索不到关键约束，反而增加噪声。  
   建议保持单文件不超过 80-120 行，超过就拆分。

2. **敏感信息全量写入**  
   把 token、密码、内网拓扑直接写进 USER.md，随后被插件全量读取。  
   敏感信息应放环境变量或 secret 管理器，USER.md 只写“如何获取”，不写值本身。

3. **格式不受约束**  
   不同人写不同格式，Agent 解析不了。  
   建议统一使用 YAML frontmatter 加固定 section，并提供模板。

4. **写一次永不更新**  
   环境变了，USER.md 没变，Agent 仍按旧约束执行。  
   建议每两周或在重大环境变更后检查一次，并用 git 记录修改原因。

5. **误用于动态状态**  
   把“当前正在开发 X 功能”写进 USER.md，很快就过期。  
   动态任务状态应放任务文件或 issue，USER.md 只放长期稳定事实。

## 可复用建议

- **按角色拆分**：`USER.md` 放全局，`USER.work.md`、`USER.home.md` 按场景加载；
- **控制加载粒度**：优先注入 frontmatter 和 `Constraints`，不要全文塞入；
- **版本化管理**：放在 dotfiles 仓库，用 commit message 说明变更原因；
- **与 MCP 解耦**：让 Agent 通过工具按需读取，而不是把内容复制进每轮对话；
- **设置确认机制**：关键任务前要求 Agent 复述约束，避免“读过了但没执行”。

## 总结

USER.md 的本质，是给 Agent 建一层“稳定的用户事实层”。它不是万能提示词，而是让 Agent 在正确的时间、以正确的方式，获取关于你的关键上下文。

在 OpenClaw 这类强调插件与自动化的环境里，能自动读、能按需取、能版本化的 USER.md，比一段很长的自我介绍更有价值。先把边界和禁忌写清楚，再谈效率提升，往往更容易复现。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/ffb0a69b86cc1d14.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/7a67e445f6e13fa3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-24/f78ffac2774bdc2d.png)

