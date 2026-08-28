---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 35137
source: 综合讨论
publishedAt: 2026-08-29
---

## 背景：能力越堆越多，上下文先撑不住

在 OpenClaw 里做 Agent 能力扩展时，最容易走的路是把所有工具、脚本、外部服务说明一次性写进 system prompt，或者启动时就全量注册 MCP、插件和命令。前期只有两三个能力时不觉得有问题，一旦能力数量到十几个、几十个，工程问题会集中爆发。

最常见的现象是：

- system prompt 越来越长，模型开始忽略中间段落或误用工具；
- token 成本上升，但任务质量没有同步提高；
- 工具描述之间互相干扰，模型在相似工具之间乱选；
- 初始化变慢，每次起会话都要加载一堆根本用不上的能力；
- 权限面过大，一个只做文本任务的会话也带着文件删除、外部请求等权限。

这本质上是“全量加载”反模式。OpenClaw 的 Skills 机制解决的就是这个问题：把能力拆成独立单元，只有任务需要时才加载。

## 问题：全量加载不是能力问题，是架构问题

很多实践者把“Agent 能力强”理解为“把所有能力都挂上去”。但在实际运行中，模型能有效利用的上下文和工具数量是有限的。与其说模型需要更多工具，不如说它需要一个清晰的加载入口。

Skills 机制的思路更接近操作系统的按需分页，或插件系统的懒加载：

1. 启动时只加载基础路由能力和 Skill 索引；
2. 收到用户任务后，根据意图匹配对应 Skill；
3. 加载该 Skill 的指令片段、工具定义、脚本路径和权限；
4. 任务结束或上下文切换后卸载，避免污染后续对话。

这样做的好处不只是省 token，更重要的是让模型在特定任务下只看到相关上下文，降低误判。

## 做法：把能力拆成可发现、可加载的 Skill

### 1. 定义一个 Skill 的最小结构

在 OpenClaw 中，一个 Skill 可以是一个独立目录，里面至少包含一个 manifest 文件和可选 prompt、脚本、工具定义。例如：

```yaml
# skills/pdf-extract/skill.yaml
id: pdf-extract
name: PDF 文本与表格提取
description: 从 PDF 中提取文本、表格和页码
triggers:
  - pdf
  - 提取表格
  - extract text
tools:
  - read_pdf
  - extract_table
permissions:
  - file_read
  - workspace_write
prompt: prompt.md
```

关键是 `description` 要写清楚“什么时候有用、什么时候没用”，而不是只写功能名。`triggers` 用于规则召回，但不要只依赖触发词。

### 2. 建立轻量索引

OpenClaw 启动时只扫描各 Skill 的 manifest，生成一个很薄的索引，不加载实际工具。索引内容大致是：

```json
{
  "id": "pdf-extract",
  "summary": "提取 PDF 中的文本、表格和页码",
  "triggers": ["pdf", "提取表格", "extract text"]
}
```

这个索引会作为路由信息注入基础 system prompt，让模型知道“存在哪些可选能力，但当前未加载”。如果模型完全不知道有哪些 Skill 可选，它就不会主动请求加载，这是很多人失败的第一站。

### 3. 匹配与加载

实际匹配可以采用“规则召回 + 模型确认”两层。规则层先根据关键词或 embedding 召回一两个候选，再让模型判断是否需要加载。伪代码大概是这样：

```python
def match_and_load(user_input, session):
    candidates = skill_index.match(user_input)
    if len(candidates) == 1 and candidates[0].score > threshold:
        skill = candidates[0]
        session.load_skill(skill)
        session.log("skill_loaded", skill.id)
        return skill
    return None
```

加载时把 Skill 的 prompt 片段追加进会话，注册该 Skill 的工具定义，并注入对应的权限边界。如果 Skill 有依赖其他 Skill，应一次性加载完整，避免半初始化。

### 4. 生命周期管理

任务边界通常可以通过会话状态判断。比如用户切换到一个新主题、调用 `skill_unload`、或一定轮次内没有使用该 Skill 的工具，就可以卸载。卸载时要回收工具定义，避免后续对话误用。

## 踩坑点

**触发词过宽。** `report`、`data`、`export` 这类词很容易同时命中多个 Skill，导致每次会话都误加载。触发词应该具体到动作对象，比如 `pdf report`、`sales data export`，并且配合模型二次确认。

**只加载工具，不加载运行条件。** Skill 经常依赖环境变量、密钥、外部文件或前置脚本。manifest 里必须声明这些依赖，否则任务跑到一半才报配置缺失。

**模型不知道未加载能力的存在。** 如果不在路由层暴露 Skill 索引，模型不会主动说“我需要加载 PDF 能力”。这是最容易忽视的一点。

**多个 Skill 的指令互相冲突。** 两个 Skill 都定义了输出格式或操作规范，同时加载后会互相干扰。需要设定优先级，或者在加载时做冲突检查。

**热加载污染会话状态。** 同一个 session 里先加载了 A，再加载 B，A 的工具和指令仍然存在，后续任务可能误用。建议任务级隔离，或给 Skill 设置 idle TTL。

**版本更新影响运行中任务。** Skill 目录直接改内容，可能导致正在执行的会话出现行为不一致。可以引入版本号锁定，运行中使用固定版本。

## 可复用建议

先不要一上来做几十个 Skill。用 3 到 5 个高频场景跑通加载、匹配、卸载全流程，再扩展。

给所有 Skill 加日志：加载时间、触发原因、加载结果、卸载时间。没有日志，排查误加载基本靠猜。

把 Skill 当成代码仓库管理，而不是简单堆脚本。加版本、加测试、做 review，尤其是涉及外部 API 或文件写操作的 Skill。

权限按 Skill 声明，按需授权。不要默认给所有 Skill 开放工作区读写权限。

最后，避免把外部不可控内容直接塞进 system prompt。Skill 的 prompt 应该经过审查，尤其是来自第三方或动态生成的部分。

## 总结

OpenClaw 的 Skills 机制不是简单地把工具打分分类，而是把 Agent 能力从“全量暴露”改成“路由 + 懒加载”。工程收益体现在三个方面：上下文更干净、权限面更小、扩展能力更可控。

真正落地时，关键不在 Skill 数量，而在索引是否清晰、触发语义是否准确、生命周期是否可管理。先把这三个基础打牢，再谈规模化。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/b25028ce660d417a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/aa96420eb9a4471f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/6c4963adff569acd.png)

