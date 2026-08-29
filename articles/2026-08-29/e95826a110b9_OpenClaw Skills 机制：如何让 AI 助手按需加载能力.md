---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 35208
source: 综合讨论
publishedAt: 2026-08-29
---

## 背景

在 OpenClaw 这类 Agent 项目里，AI 助手要处理的事情越来越多：文件解析、SQL 查询、浏览器自动化、消息推送……最常见的做法是每增加一个能力，就注册一个常驻工具或插件。结果系统提示和工具定义越来越长，上下文窗口被大量非当前任务所需的描述占满。模型推理变慢、token 成本上升，甚至出现工具选择混乱——明明用户只是想查个 PDF，模型却去调用数据库插件。

这暴露了一个核心问题：我们习惯把“能力”做成常驻，而不是按需加载。

## 问题

全部能力常驻会带来几个明显后果：

- 工具描述互相干扰，模型选错工具的概率上升；
- 上下文预算被低价值能力占用，影响长对话表现；
- 插件依赖和权限范围扩大，增加安全风险；
- 每加一个功能，维护成本不是线性增长，而是叠加在全局复杂度上。

因此需要一种 Skills 机制：把能力拆成可发现、可加载、可回收的单元，在需要时才注入完整指令和工具实现。

## 做法 / 步骤

在 OpenClaw 中落地 Skills 机制，可以分成五步。

### 1. 定义 Skill 清单

每个 skill 至少包含这些字段：

```yaml
skills:
  - name: pdf-extract
    description: Extract text and tables from PDF files
    triggers: [pdf, extract, parse]
    entrypoint: skills/pdf_extract.py
    dependencies: [pypdf]
    permissions: [read:files]
    timeout: 60
```

关键是把“元信息”和“实现体”分开。元信息负责让模型判断是否需要加载，实现体是加载后真正执行的内容。

### 2. 只注入索引，不注入全文

主 prompt 中不要放每个 skill 的完整指令，只放索引：

```text
Available skills:
- pdf-extract: Extract text and tables from PDF files. Trigger when user mentions PDF or text extraction.
- sql-query: Run read-only SQL queries. Trigger when user asks about database data.
```

模型看到的是简短索引，不会一上来就被大量工具描述淹没。

### 3. 命中后加载

当用户意图命中 triggers，或者模型根据索引判断需要某个 skill 时，再调用 loader 读取完整指令、注册工具、安装依赖。加载动作最好记录以下信息：

- skill_id
- 触发词 / 判断依据
- 加载耗时
- token 预估消耗
- session_id

这为后续优化提供数据。

### 4. 隔离执行

每个 skill 在独立 workspace、namespace 或容器中执行，避免污染全局上下文。比如 `sql-query` 只拿到只读数据库连接，`pdf-extract` 只拿到文件读取权限。skill 之间不共享全局状态。

### 5. 任务完成后卸载

任务结束就卸载 skill，释放 token、内存和临时文件。卸载要彻底，不能只从 prompt 里移除定义，还要清理运行时状态。

## 踩坑点

实际落地时，最容易出问题的不是加载逻辑，而是元信息设计。

**触发词太宽泛**  
比如把 `read` 作为 `file-reader` 的触发词，会导致任何包含“读取/查看/打开”的请求都可能触发加载，最后几乎每次对话都加载一遍。触发词应当具体到动词 + 对象，例如 `pdf`、`extract table`。

**描述太抽象**  
如果 skill 描述只写“处理文件”，模型无法判断什么时候该用，会出现漏触发或误触发。描述要明确输入、输出和适用边界。

**只加载不回收**  
有些实现加载很积极，但不做卸载，上下文逐渐膨胀，最后还是回到常驻的老路。加载和卸载必须成对设计。

**依赖冲突**  
多个 skill 可能需要不同版本的同一个库。全局安装容易冲突，建议每个 skill 使用独立虚拟环境或容器，并锁定版本。

**权限过大**  
不要为了省事直接给 skill 开放全部权限。默认最小权限，按需临时授权。

**缺乏可观测性**  
没有加载日志和成本统计，就不知道哪些 skill 命中率低、哪些消耗高，优化无从下手。

## 可复用建议

- 每个 skill 保持单一职责，描述控制在 1～2 句，明确边界。
- triggers 同时包含正向触发和负面示例，降低误判。
- 用 loader 记录加载事件，定期统计命中率和 token 成本。
- skill 之间通过显式接口通信，不要共享全局变量。
- 依赖隔离、版本锁定，有条件直接容器化。
- 给每个 skill 设置默认 timeout 和权限白名单。

## 总结

Skills 机制本质上是给 AI 助手做“延迟加载”和“上下文预算管理”。它不是为了增加更多功能，而是让已有能力在正确的时机出现。OpenClaw 场景中，先把元信息、生命周期、隔离和可观测性做好，比堆一堆 skill 更重要。按需加载一旦稳定，token 成本、工具冲突和推理稳定性都会明显改善。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/1709714da9fd3697.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/cb0c44ad868c2798.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/fd4de9716b87dd7f.png)

