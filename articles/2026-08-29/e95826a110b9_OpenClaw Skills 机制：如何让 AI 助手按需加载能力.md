---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 35158
source: 综合讨论
publishedAt: 2026-08-29
---

# OpenClaw Skills 机制：如何让 AI 助手按需加载能力

## 背景
在 OpenClaw 的 Agent 项目中，我们习惯把能力拆成若干工具、MCP 服务器或插件。但当工具数量从个位数涨到几十个时，常驻加载的代价就很明显：每次会话启动都要初始化所有工具，上下文里塞满工具描述，token 消耗持续上升；有些工具只在特定场景用到，却一直占用连接和内存。

我自己的教训是：一个客服 Agent 接了订单、物流、发票、知识库、CRM 等 20 多个 MCP 工具，结果首轮响应延迟增加了 40%，而且经常因为工具描述互相干扰导致模型选错工具。后来引入了 Skills 机制，把不常用的能力按需加载，问题才缓解。

## 问题
按需加载的核心矛盾是：加载太晚，用户等待；加载太早，又回到全量占用。需要解决四件事：
- 如何声明一个 skill 的能力边界和触发条件
- 如何在不全量加载的情况下发现并命中 skill
- 加载后如何注入当前会话，让模型知道新工具可用
- 如何安全卸载，避免残留

## 做法/步骤

### 1. 定义 Skill Manifest
每个 skill 一个目录，包含 `manifest.yaml` 和入口文件。manifest 描述能力、触发条件、权限和生命周期。
```yaml
id: invoice-query
description: 查询发票开具状态
triggers:
  - invoice
  - receipt
  - 发票
entry: invoice_query.py
permissions:
  - read:invoice
  - call:erp-api
ttl: 300
dependencies:
  - package: requests>=2.28
```
`triggers` 用于触发匹配，`ttl` 控制空闲卸载时间，`permissions` 在加载时展示给用户。

### 2. 配置 Skill Registry
在 OpenClaw 配置中指向 skills 目录：
```json
{
  "skills": {
    "dir": "./skills",
    "auto_discover": true,
    "match_strategy": "vector",
    "min_score": 0.65,
    "default_ttl": 180,
    "max_loaded": 5
  }
}
```
`match_strategy` 可以是 `keyword` 或 `vector`，`max_loaded` 限制同时加载数量，防止内存膨胀。

### 3. 触发加载
OpenClaw 收到用户消息后，先用轻量级匹配器扫描 `manifest` 的 `triggers` 或向量索引，对候选 skill 打分。超过 `min_score` 的 skill 进入待加载队列。
```python
skills = registry.match(user_query)
for skill in skills:
    if not skill.is_loaded():
        skill.load(permissions=request_permissions(skill))
        session.register_tool(skill.tools)
```
加载后，skill 暴露的工具会动态注入到当前会话的工具列表，模型在下一轮就能调用。

### 4. 生命周期管理
每个 skill 实现 `load()`、`unload()` 和可选的 `health_check()`。会话结束或 TTL 到期后，OpenClaw 调用 `unload()` 释放资源。建议用引用计数，避免多个并发会话同时卸载。

## 踩坑点
- **触发词太宽泛**：早期用 `read` 做 trigger，几乎每条消息都触发加载，导致每次都要调权限弹窗，影响体验。后来改成向量匹配 + 关键词组合，误报明显减少。
- **依赖未声明或版本冲突**：一个 skill 用了特定版本 SDK，加载时才发现环境里没有，报错后整个会话中断。现在 manifest 强制声明依赖，加载前先验证。
- **权限提示不清晰**：自动加载时模型替用户做决定，用户对突然出现的权限请求感到困惑。建议在加载前明确提示“是否加载 xxx skill，需要以下权限……”，并允许拒绝。
- **并发加载竞争**：多个 skill 同时操作同一个配置文件或端口，产生竞态。用文件锁或单例加载器解决。
- **卸载不干净**：有些 skill 注册了全局事件监听或后台线程，卸载后还在跑。要求每个 skill 必须实现 `teardown()`，并在卸载时做资源追踪。

## 可复用建议
1. **保持 manifest 最小化**：只写必备字段，不要为了灵活性加一堆配置，增加维护成本。
2. **会话级缓存**：同一 skill 在会话内只加载一次，用 TTL 或引用计数控制卸载，避免反复初始化。
3. **可观测性**：记录每个 skill 的加载耗时、触发命中率、失败率，定期调整 `min_score` 和 `max_loaded`。
4. **版本锁定**：在 manifest 中明确依赖版本范围，避免环境漂移导致加载失败。
5. **灰度发布**：新 skill 先在小流量或测试会话中启用，观察稳定性和误触发情况再放开。

## 总结
Skills 机制不是简单把工具变成懒加载，而是一套完整的能力生命周期管理：声明、发现、加载、注入、卸载。它让 Agent 在保持轻量上下文的同时，按需获取专项能力。对 OpenClaw/Agent 实践者来说，合理的 Skills 设计能显著降低资源占用和上下文噪声，值得在项目早期就引入。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/bc5d5882e0cfc4c0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/f7068d1cdc66846b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/f5f3b2626d681b16.png)

