---
title: OpenClaw Skills 按需加载实践：别让 AI 助手背着一整包插件上班
feedId: 33948
source: 综合讨论
publishedAt: 2026-08-21
---

## 背景

OpenClaw 作为个人 AI 助手，真正的价值不在于内置多少模型能力，而在于能否把外部工具、自动化脚本、业务 API 变成可组合的“技能”。随着脚本越积越多，一个常见做法是把所有 skill 在启动时全部注册进上下文。短期能用，但很快暴露问题：上下文窗口被描述文本占满、工具选择准确率下降、启动变慢、低权限 skill 长期驻留增加风险。

## 问题

全量加载主要有三个代价：

1. **上下文污染**：每个 skill 的描述、参数、示例都进 system prompt，实际对话空间被挤占。
2. **选择退化**：几十个工具摆出来，模型更容易选错或漏选。
3. **维护成本高**：一个 skill 报错可能影响全局，排查时难以定位。

理想状态是：AI 只看到“目录”，真正用到某个能力时再加载实现细节。

## 做法/步骤

下面是在 OpenClaw 环境里落地按需加载的一套可行流程。

### 1. 给每个 skill 建独立目录

```text
skills/
  github-issue/
    manifest.yaml
    handler.py
    README.md
  pdf-summary/
    manifest.yaml
    handler.py
```

每个 skill 保持单一职责，目录名即标识。

### 2. 用 manifest 描述触发条件

manifest 里只放轻量元数据，不放大段实现说明：

```yaml
name: github-issue
description: 创建 GitHub issue
triggers:
  - "create issue"
  - "github issue"
permissions:
  - repo.write
  - network
entrypoint: handler.py
```

关键点：`description` 要短，只用于模型判断“何时调用”；`triggers` 帮助路由；`permissions` 用于做权限闸门。

### 3. 启动时只扫描 manifest，不加载实现

OpenClaw 启动时遍历 skills 目录，读取所有 manifest，生成一个技能索引放进上下文。这部分内容要压缩成表格，例如：

```text
- github-issue: 创建 GitHub issue [repo.write, network]
- pdf-summary: 总结 PDF [file.read]
```

模型只看到这一层。

### 4. 命中时再动态加载

当用户请求匹配某个 skill 时，再根据 manifest 的 `entrypoint` 加载实际代码。加载后执行，完成后销毁临时运行时。可以理解为“即用即载”。

### 5. 加入超时与资源限制

动态加载必须配超时，否则一个失控 skill 会拖慢整个会话。建议给每个 skill 设置执行超时、内存上限、输出长度截断。

## 踩坑点

1. **manifest 与实现不同步**：改了代码没改 triggers，模型永远路由不到。建议在 CI 或本地 hook 里校验 manifest 字段完整、入口文件存在。
2. **命名冲突**：多个 skill 都用 `handler.py` 不重要，但入口必须用相对 skill 目录解析，不要依赖全局搜索路径。
3. **动态加载的依赖缺失**：按需加载意味着启动时不会提前发现缺依赖。可在加载失败时返回结构化错误，并提示用户安装依赖，而不是静默降级。
4. **权限声明过宽**：manifest 里写 `permissions: all` 会让按需加载失去意义。建议最小权限声明，并在执行前做一次校验。
5. **热加载不生效**：改了 skill 后没有重新扫描索引，导致新触发器不生效。开发期可以加一个 `skills/reload` 命令手动刷新。

## 可复用建议

- **分层**：把 skill 分为“元数据层”和“实现层”，元数据常驻，实现按需加载。
- **用触发词而不是让模型自己猜**：显式 triggers 能降低路由错误率。
- **给每个 skill 写一个最小测试**：只验证 manifest 可解析、入口可加载、参数校验通过。
- **错误要可见**：skill 调用失败时，返回错误摘要给模型，让它能解释给用户，而不是把堆栈打进聊天。
- **定期清理**：不再用的 skill 及时下线，避免索引膨胀。

## 总结

OpenClaw Skills 的按需加载，本质上是一个“注册表 + 懒加载 + 权限闸门”的组合。它不追求把所有能力一次性塞给模型，而是让模型在需要时找到对的能力。工程实现不复杂，真正需要花力气的是 manifest 设计、错误处理和权限约束。做到这三点，AI 助手才能既轻量又可扩展。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/b58b73d758057eb9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/07037f5808f0bfd4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/c77c61abd4b383aa.png)

