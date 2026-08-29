---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力
feedId: 35203
source: 综合讨论
publishedAt: 2026-08-29
---

## 背景：能力越多，助手越容易变笨

在 OpenClaw 里接入 MCP、插件和内部自动化工具后，一个常见问题是：我们把所有能力都挂进系统，反而让助手上下文膨胀、响应变慢、工具选择准确率下降。尤其是只会在少数场景用到的能力，比如数据库排障、发布回滚、某个业务域查询，如果每次都全量注入提示词和工具定义，成本很高。

OpenClaw Skills 机制解决的正是这个问题：把能力拆成独立 skill，系统默认只保留轻量索引，在识别到相关任务时再按需加载完整提示词、工具和权限。这样常驻上下文只保留“目录”，而不是“全书”。

## 问题：为什么不能无脑全量加载

全量加载至少有四个代价：

1. Token 成本：不常用的工具定义和说明也会占用上下文窗口。
2. 选择干扰：工具过多时，模型更容易选错或漏选。
3. 启动变慢：每个工具、MCP client 都初始化会拖慢会话启动。
4. 边界模糊：多个 skill 同时在线时，权限、环境变量和命名容易互相污染。

按需加载不是简单延迟，而是要有明确的触发边界和生命周期管理。

## 做法：以 skill 为单位拆分和加载

一个可落地的目录结构如下：

```text
skills/
  sql-query/
    manifest.yaml
    prompt.md
    tools.py
  incident-response/
    manifest.yaml
    prompt.md
    tools.py
```

每个 skill 用 manifest 描述自己，但不直接加载完整内容。示例：

```yaml
name: sql-query
description: 查询只读数据库，生成分析结果
triggers:
  - 查数据库
  - 跑 SQL
  - 数据核对
negative_triggers:
  - 修改表结构
  - 写库
entrypoint: prompt.md
tools:
  - query_db
ttl: 300
permissions:
  - database.read
```

系统启动时只扫描 manifest，建立索引。会话中命中触发条件时，再加载 `prompt.md`、注册 `query_db` 工具、注入相关权限。会话结束或触发域切换后，按 TTL 或显式卸载。

触发方式建议组合使用：

- 关键词/意图匹配：适合用户口语化指令。
- 命令式触发：例如明确说“加载 sql-query”。
- 失败回退：主工具调用失败后，再尝试加载对应 skill。

## 踩坑点

1. **描述不当导致误触发**  
   manifest 里 description 和 triggers 写得太宽，会让 skill 频繁加载；写得太窄，又很难命中。建议同时写 `negative_triggers`，明确“什么情况不要加载”。

2. **路径解析不一致**  
   `entrypoint` 和工具文件如果混用绝对路径、相对路径，换环境后容易加载失败。建议所有资源路径都相对 skill 根目录解析。

3. **卸载不干净**  
   只移除了提示词，没有注销工具或清理环境变量，会留下“幽灵能力”。加载和卸载要成对实现，最好有 unload 钩子。

4. **多个 skill 冲突**  
   两个 skill 注册同名工具或修改同一个环境变量。可以通过命名空间、加载锁和依赖声明来控制并发和冲突。

5. **权限一次给太满**  
   按需加载不等于把该域全部权限都激活。仍应遵守最小权限，加载时只授予当前任务需要的权限。

6. **缓存未失效**  
   manifest 更新后，旧索引仍被缓存，导致新触发词不生效。建议给 manifest 加版本号或 content hash，加载前校验。

## 可复用建议

- 索引摘要与完整资源分离，启动只读摘要。
- 每个 skill 必须有正向和负向触发示例，减少误加载。
- 为每个 skill 写一个 smoke test：加载、执行最小任务、卸载三步骤。
- 记录加载原因和耗时，方便排查“为什么这次多加载了某个 skill”。
- 限制同时加载的 skill 数量，避免一次命中多个大 skill 又把上下文撑爆。
- manifest 版本化，避免热更新后出现新旧逻辑混用。

## 总结

OpenClaw Skills 机制的本质，是用轻量索引替代全量常驻，用显式触发替代无条件注入。它适合 MCP 工具多、插件可选、自动化场景分散的情况。落地时最难的不是加载，而是把边界描述清楚、把卸载做干净。做好这两点，能力扩展才不会变成上下文负担。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/f77781a48bff3b7e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/e9beeeb400c294de.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/57a7310834ffb7d2.png)

