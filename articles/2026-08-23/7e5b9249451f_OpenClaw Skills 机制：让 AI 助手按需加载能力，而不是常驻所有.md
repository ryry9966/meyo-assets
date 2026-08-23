---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力，而不是常驻所有插件
feedId: 34298
source: 综合讨论
publishedAt: 2026-08-23
---

# OpenClaw Skills 机制：让 AI 助手按需加载能力，而不是常驻所有插件

## 背景

在 OpenClaw 里接了多个 MCP、自定义脚本和插件之后，最明显的变化不是能力变强，而是每次对话越来越慢。系统提示里堆着所有工具的说明，模型经常在不需要的时候调用无关工具，Token 消耗也居高不下。后来把“常驻插件”改成“按需加载的 Skills”，上下文才干净下来。

这篇文章记录我在 OpenClaw 上落地 Skills 机制的过程，重点不是概念，而是可复用的工程做法。

## 问题

原来的做法是：所有能力都在启动时注册，不管这次会话是否需要。带来的问题包括：

- 系统提示膨胀，很多指令与当前任务无关。
- 模型容易被多余工具干扰，选错调用路径。
- MCP 连接长期保持，资源占用高。
- 工具名称冲突时，需要在全局范围内协调，维护成本大。

Skills 机制的核心思路是：启动时只读取轻量元数据，会话中根据用户输入命中后再加载完整指令、工具或 MCP 连接，任务结束或闲置后卸载。

## 做法与步骤

我在 OpenClaw 里的 Skills 目录结构如下：

```text
skills/
  summary/
    SKILL.md
  data-query/
    SKILL.md
    scripts/
      query.py
  jira/
    SKILL.md
    mcp.json
```

每个 `SKILL.md` 顶部用 frontmatter 声明元数据：

```yaml
---
name: summary
description: Summarize long documents or transcripts
triggers:
  - summarize
  - tl;dr
  - 摘要
tools:
  - name: summarize_text
    type: mcp
    ref: summary-mcp
max_tokens: 1200
unload_after_idle: 10m
---
```

**第一步：启动时只扫描元数据。**

加载器遍历 `skills/` 目录，只读取 frontmatter，不加载正文，也不建立 MCP 连接。这样启动成本很低。

**第二步：会话内做轻量匹配。**

对用户输入做两层匹配：

1. 精确触发词命中：`triggers` 里的词出现在输入中。
2. 描述相似度兜底：用简单的文本相似度判断用户意图是否接近某个 skill 的 `description`。

匹配逻辑简化如下：

```ts
const hits = registry.filter(skill => {
  const hitTrigger = skill.triggers.some(t => input.includes(t));
  const hitDesc = similarity(input, skill.description) > 0.6;
  return hitTrigger || hitDesc;
}).slice(0, 2);
```

**第三步：命中后加载。**

将 `SKILL.md` 正文注入到当前会话的临时作用域，同时按需注册工具。如果是 MCP 类型，建立连接并注册对应工具；如果是本地脚本，包装成可调用工具。

**第四步：闲置卸载。**

如果一段时间没有调用该 skill 的工具，或者当前任务结束，就卸载指令、注销工具并断开 MCP 连接，释放上下文和资源。

## 踩坑点

**1. 触发词太宽泛，频繁误加载。**

比如把触发词设置成 `data`，几乎每轮对话都会命中。触发词最好用动词短语或带领域词，比如 `query database`、`create ticket`，而不是单个高频词。

**2. 卸载不干净，上下文残留。**

如果只是把 skill 指令追加到系统消息，卸载时很难精准删除。有的模型对多段系统消息支持不一致，容易残留旧指令。建议在 session 层维护一个 active skills 列表，每次重建当前会话的 system prompt，而不是做字符串增删。

**3. MCP 连接生命周期没管好。**

Skill 卸载了，但 MCP 连接还在，工具仍然注册在模型侧。必须在卸载钩子里显式 `unregister` 工具并断开连接，否则会出现“已卸载但仍然可调用”的诡异问题。

**4. 工具命名冲突。**

不同 skill 提供同名工具时，后加载的会覆盖先加载的。需要在注册时加命名空间前缀，例如 `summary.summarize_text`，并在 `SKILL.md` 里写明模型应使用的完整工具名。

**5. 缓存旧版本，改完不生效。**

开发阶段经常改 `SKILL.md`，但加载器缓存了 manifest，导致新触发词不生效。可以按文件 mtime 或内容 hash 做失效检查，开发环境直接关闭缓存。

## 可复用建议

- **Skill 粒度控制在单任务**，不要把一个领域的全部能力塞进一个 skill。
- **SKILL.md 正文保持精简**，目标注入 Token 不超过 1500，否则加载收益会下降。
- **设置同时加载上限**，比如一次最多 3 个 skills，避免上下文再次膨胀。
- **支持负触发词**，例如 `notWhen: [不需要摘要]`，减少误加载。
- **记录加载/卸载日志**，包含命中原因和耗时，方便排查匹配质量。
- **对 MCP 连接做连接池或懒加载**，不要每次命中都新建连接。
- **依赖声明要写清楚**，如果某个 skill 依赖另一个 skill 的工具，需要在 manifest 里声明 `dependencies`，否则按需加载可能缺少前置能力。

## 总结

OpenClaw Skills 机制不是简单的“把提示词拆成文件”，而是一套轻量注册、按需注入、生命周期管理的加载策略。它解决的问题不是能力不足，而是能力全量常驻带来的上下文污染和资源浪费。

落地时优先级应该放在：合理的触发词设计、干净的卸载机制、明确的工具命名空间。这三点做好之后，Skills 机制才能真正让 AI 助手在需要时变强，在不需要时保持轻量。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/44e0a5e6f93f836c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/03a2562ad3979e92.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-23/68fbaed06f084fd6.png)

