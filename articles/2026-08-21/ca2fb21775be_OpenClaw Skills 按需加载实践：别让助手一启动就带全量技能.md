---
title: OpenClaw Skills 按需加载实践：别让助手一启动就带全量技能
feedId: 33980
source: 综合讨论
publishedAt: 2026-08-21
---

## 背景

OpenClaw 的 Skills 机制容易做成“启动即全量加载”：把 yaml/js 全部塞进上下文。技能少还行，超过 20 个后，上下文消耗、启动延迟、工具冲突都会明显出现。按需加载不是额外优化，而是多技能工程的基本边界。

我维护的一个 OpenClaw 实例跑过 30+ skills，包括通知推送、文件整理、Git 操作、网页抓取等。最开始默认全量加载，平均首包延迟多出 2-4 秒，部分 skill 的触发词还会互相干扰。后来改成按需加载，上下文占用下降约 40%，技能冲突基本消失。

## 问题拆解

全量加载主要带来四类问题：

1. **上下文污染**：所有 skill 的描述、参数、示例进入系统提示，影响主任务判断。
2. **启动变慢**：每个 skill 初始化、校验 schema、注册工具都需要时间。
3. **权限与副作用风险**：部分 skill 有写操作或网络访问，默认可用会增加误触发概率。
4. **命名冲突**：多个 skill 可能注册同名工具或相似触发词，实际执行不稳定。

按需加载要解决的核心是：**默认只给索引，用户或任务明确需要时再挂载。**

## 做法步骤

### 1. 每个 skill 独立目录，manifest 只声明元信息

```
skills/
  github-helper/
    manifest.yaml
    skill.js
  rss-digest/
    manifest.yaml
    skill.js
```

manifest 建议保持最小化：

- name
- version
- description：一句话，供索引匹配
- triggers：触发词/意图
- permissions：所需权限
- deps：依赖的其他 skill，可选但建议保留

### 2. 启动时只加载 manifest 索引，不加载实现

OpenClaw 启动时扫描 `skills/*/manifest.yaml`，生成轻量索引。索引只包括 name、description、triggers、permissions，不读取 skill.js。这样系统提示里只有技能目录，而不是全量技能内容。

### 3. 命中后再加载实现

对话或任务进来时，用索引做匹配。命中 triggers 或语义相似度超过阈值后，再读取 skill.js，注册工具/指令。未命中的 skill 不加载。

### 4. 设置作用域和卸载策略

按需加载后要明确 skill 何时生效、何时卸载。可以按 session、task 或调用次数。例如 github-helper 在用户说“提交到 GitHub”后加载，任务结束或新 session 时卸载，避免残留工具定义。

### 5. 显式声明权限与前置条件

manifest 里的 permissions 可以区分 read/write/network/exec。索引阶段就过滤掉用户未授权的 skill，减少误触发。

## 踩坑点

- **只匹配关键词，不匹配语义**  
  “把这次改动推到仓库”不会命中“git push”。需要 triggers 加别名，或者用 embedding 做轻量语义匹配。初期用关键词 + 同义词表即可，成本低。

- **manifest 暴露过多细节**  
  description 写太长，索引反而变重。建议控制在 40-80 字，只写“能做什么、何时用”。

- **卸载不干净**  
  skill 注册了工具但会话切换后未卸载，导致旧上下文仍可调用。需要在 loader 里记录注册的 tool name，卸载时逐一反注册。

- **依赖未声明**  
  skill A 依赖 skill B 的输出，但 manifest 没写 deps。按需加载 A 时 B 没加载，运行时报错。建议 deps 必填，加载器递归加载依赖。

- **并发冲突**  
  两个 skill 都想修改同一个文件或注册同名工具。manifest 增加 namespace，工具名统一为 skillName.toolName，减少冲突。

## 可复用建议

1. 默认 deny，显式 enable。
2. skill 必须单一职责，一个 skill 不要同时做浏览器抓取和数据库写入。
3. manifest 即契约，字段保持稳定，别往里塞执行逻辑。
4. 建立 smoke test：加载一个 skill 后只跑最小示例，确认工具注册、输出、卸载正常。
5. 版本锁定 manifest，skill 升级要能回滚。

## 总结

OpenClaw Skills 机制的关键不是“能不能加载”，而是“什么时候加载、什么时候卸载”。把 manifest 当索引、实现当懒加载资源、权限和依赖显式声明，能让多技能助手在工程环境中保持稳定。对长期维护来说，这比堆技能数量更重要。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/bc41d63ad53a7a80.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/a7362471094e1aae.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-21/a127fe017bf3d940.png)

