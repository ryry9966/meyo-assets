---
title: Agent 安全红线：防止 AI 在社区泄露工作流和内部信息
feedId: 34025
source: 综合讨论
publishedAt: 2026-08-21
---

## 背景

在 OpenClaw 这类 Agent 编排环境里，社区自动化越来越多：自动回复 issue、整理讨论摘要、发布更新说明、生成公开文档。问题在于，这些 Agent 常常不是从零配置的，而是复用了内部工作流里已有的 system prompt、MCP 工具和插件示例。

于是会出现一种尴尬情况：Agent 在社区里表现得很“能干”，但它能说出内部目录结构、内网域名、工单处理 SOP，甚至把环境变量或 token 片段带进公开输出。这不是模型幻觉，而是典型的上下文污染。

## 问题：泄露通常不来自恶意，而来自三层污染

1. **上下文污染**：把内部 SOP、路径、账号规则直接写进社区 Agent 的 system prompt，模型会把这些当作可引用知识。
2. **工具返回污染**：MCP 工具返回过大的 JSON、错误堆栈或数据库字段，模型在总结时原样带出。
3. **配置示例污染**：分享插件或 Agent 配置时，只截取局部，但模型补全或复述时带出了未脱敏变量。

真正危险的不是 AI 主动“背叛”，而是我们给了它太多不该出现在公开场景下的上下文。

## 做法/步骤

### 1. 公共 Agent 使用独立身份和最小 system prompt

不要直接复用内部 Agent 配置。社区 Agent 应该有一个专用身份，只描述公开任务边界。

```yaml
public_agent:
  system_prompt: |
    You are a community assistant.
    Allowed output: issue summary, reproduction steps, public doc links.
    Never include: file paths, hostnames, tokens, internal SOP names.
    If unsure, reply: "I cannot share that."
```

内部工作流信息不要写在这个 prompt 里。需要动态数据时，通过工具返回值注入，而不是预置在上下文中。

### 2. 工具返回做最小化和字段白名单

MCP 工具很容易返回过多信息。建议在工具层或 wrapper 层做裁剪：

```ts
function sanitizeToolOutput(text: string) {
  return text
    .replace(/(?:[A-Za-z]:\\[\w\\-]+|\/home\/\w+\/[\w\/-]+)/g, '[PATH]')
    .replace(/(?:https?:\/\/)?[a-z0-9-]+\.internal(?:[:/][^\s]*)?/gi, '[INTERNAL_URL]')
    .replace(/(AKIA|ghp_|sk-)[A-Za-z0-9]{8,}/g, '[TOKEN]')
    .slice(0, 1200);
}
```

同时限制工具返回字段，比如只允许 `title`、`summary`、`public_url`，不要直接返回完整记录。

### 3. 加输出规则，不靠 prompt 自觉

在 Agent 输出到社区之前，加一道非模型规则过滤。可以用正则拦截常见敏感特征：

```text
(\/home\/|\/Users\/|C:\\)|\.internal\b|\.local\b|10\.\d+\.\d+\.\d+|(AKIA|ghp_|sk-)[A-Za-z0-9]{8,}
```

命中后直接 block，返回固定话术，而不是让模型判断“能不能发”。

### 4. 日志脱敏与审计

Agent 的输入输出日志要保存，但必须脱敏。建议记录 hash 后摘要，而不是明文。这样既能追溯问题，又不会让日志本身变成泄露源。

### 5. 用红队用例验证

配置完成后，手动喂入包含内部路径、内网域名、假 token 的内容，看 Agent 会不会在公开回答中带出。如果会，说明还需要收紧工具返回或输出过滤。

## 踩坑点

- **只在 prompt 里写“不要泄露”基本无效**。模型在长上下文或多轮对话中很容易忽略这类宽泛指令，必须配合输出规则和工具裁剪。
- **错误堆栈是泄露重灾区**。工具报错时，模型经常直接粘贴堆栈信息，里面可能包含路径和主机名。建议工具层catch 后只返回标准错误码。
- **分享配置时容易带出变量**。尤其是 `env` 字段、`headers`、`database_url` 这些，分享前应统一替换为占位符。
- **工具描述本身可能泄密**。MCP 工具描述里如果写了内部 API 地址或参数说明，模型可能原样复述给社区用户。描述只写公开语义。

## 可复用建议

1. **维护 public/private 两套 Agent 配置**，不要用同一份配置切换环境。
2. **敏感值一律用环境变量注入**，不直接出现在配置文件或 prompt 中。
3. **输出前过滤器独立于模型**，使用确定性规则而不是模型判断。
4. **工具返回默认最小化**，能返回摘要就不返回全文，能返回字段就不返回对象。
5. **保存脱敏审计日志**，保留 hash 和规则命中记录，方便事后排查。

## 总结

社区 Agent 的安全红线，不是在 prompt 里加一句“别泄露”，而是把边界下沉到上下文、工具、输出三层。公共身份要小，工具返回要窄，输出规则要硬。只有把这些做成可复用的工程配置，自动化才敢在公开社区里长期运行。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/d12cc609704c7da0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/36f540c906bc48a1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/bcade6615f685abf.png)

