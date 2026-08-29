---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 35312
source: 综合讨论
publishedAt: 2026-08-30
---

在 OpenClaw 里跑多 Agent 或长任务时，很容易把一堆工具、脚本、模型能力全塞进同一个上下文。刚开始方便，但代价是指令越来越长，依赖互相打架，并且每次启动都要初始化所有模块。后来我们开始用 Skills 机制做按需加载，情况才好转。

这篇文章不讨论“AI 应该有多少技能”的泛话题，只记录一个可落地的 Skills 加载实现和一些踩坑。

## 背景

OpenClaw 的 Agent 经常需要调用外部能力，比如生成报表、解析 PDF、操作浏览器。早期把这些都写成全局工具，每个请求都加载，导致两个明显问题：

1. **上下文膨胀**：几十个 skill 的描述、参数 schema 全塞进系统提示词，真正留给业务逻辑的 token 反而被挤占。
2. **依赖冲突**：一个 skill 要求 `pydantic>=2`，另一个还锁在 `1.x`；或者同一个全局环境里装了 `chromedriver` 和 `playwright`，启动耗时长，还偶尔因为版本问题崩掉。

所以目标很明确：**让 Agent 平时轻装，需要某个能力时才把对应的 skill 拉起来。**

## 问题定义

全量加载的代价不只是 token 和内存，还有安全面扩大、启动慢、错误难定位。OpenClaw 的 Agent 可能同时跑十几个任务，如果每个任务都牵着所有 skill，很快就把资源吃满。而且一旦某个 skill 有漏洞，全量加载等于把所有能力都暴露给错误输入。

## 做法/步骤

我们采用“目录即 skill”的方式，每个 skill 是一个带 manifest 的独立目录。

### 1. 定义 skill 结构

```
skills/
  pdf-export/
    skill.yaml
    main.py
    requirements.lock
  browser-automation/
    skill.yaml
    main.py
    ...
```

`skill.yaml` 只声明元数据，不写具体逻辑：

```yaml
name: pdf-export
version: 1.2.0
description: Export tabular data to PDF
triggers:
  - export pdf
  - generate report
permissions:
  - file:write:./exports
entrypoint: main.py
dependencies:
  reportlab: 4.2.2
```

关键点：`description` 和 `triggers` 是唯一被加载进上下文的信息，用来让 Agent 判断何时需要加载 skill；真正的代码和依赖不会提前进入上下文。

### 2. 注册表与匹配

主 Agent 维护一个 skill registry，启动时只扫描所有 `skill.yaml`，把 name、description、triggers 放进轻量索引。匹配方式可以简单用关键词命中，也可以用 embedding 做语义检索。我们的经验是：**关键词命中做初筛，再交给 LLM 确认是否加载**，误加载率低很多。

### 3. 懒加载与隔离

匹配到 skill 后，不直接 `import`。而是在独立 venv 或容器里安装依赖，再通过 entrypoint 执行。示例逻辑：

```python
def load_skill(name: str):
    manifest = load_manifest(name)
    if not is_loaded(name):
        env = get_or_create_venv(manifest.name, manifest.dependencies)
        module = importlib.import_module(manifest.entrypoint)
        registry.register(name, module, env)
    return registry.get(name)
```

执行完后并不立即卸载，而是放进热缓存。设置 `max_idle_time=300`，超过 5 分钟不活动再回收 venv，这样可以避免频繁创建环境的开销。

### 4. 权限限制

`permissions` 字段在运行前由策略引擎校验，不在列表里的操作直接拒绝。对于敏感操作，比如 `file:write` 到非白名单目录，会要求用户确认。

## 踩坑点

- **描述写得太宽泛**：某个 skill 的 description 是“处理文件”，结果几乎每个请求都想加载它。后来我们规定 description 必须包含具体动作和对象，比如“把 CSV 转成 PDF 报表”。
- **依赖没锁版本**：第一次用 `requirements.txt` 而不是 `requirements.lock`，有次远程装包升级了 `reportlab`，PDF 排版直接乱了。
- **卸载不干净**：有些 skill 会在模块级初始化连接池或写临时文件，直接 `sys.modules.pop` 之后状态没清理，跨会话污染数据。需要在 manifest 里定义 `cleanup` 函数，卸载时显式调用。
- **并发加载竞争**：两个任务同时请求同一个 skill，导致创建了两个 venv。后来给每个 skill 加了加载锁，已加载状态检查加在锁内。
- **缓存策略激进**：一开始永久缓存所有加载过的 skill，内存占用随任务种类线性上涨。改成 LRU + 超时回收后稳定很多。

## 可复用建议

- **manifest 做 schema 校验**：CI 里跑一个校验脚本，防止缺失 `entrypoint` 或 `permissions` 格式错误。
- **记录加载日志**：每次加载记录 skill 名、版本、耗时、触发原因。出问题时能很快定位是匹配错误还是依赖安装失败。
- **提供调试命令**：在 OpenClaw 里实现 `/skills list` 和 `/skills unload name`，方便手动检查当前已加载和强制回收。
- **给 skill 写 smoke test**：每个 skill 包里放一个最小测试，CI 跑通才允许发布。这比等线上出问题再修成本低得多。

## 总结

Skills 机制本质上就是把“能力”当插件管理：平时只暴露元数据，触发时再加载实现，用完回收。它和 MCP 不冲突，反而可以作为 MCP 之上的细粒度能力层。工程上真正花时间的不是加载逻辑，而是 manifest 质量、依赖锁定、权限边界和卸载清理。把这些做扎实，OpenClaw Agent 才能在多任务下保持轻量、稳定、可维护。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/3b3a7c37526cdedf.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/ef989939dcb1d745.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-30/9567da0b36c271af.png)

