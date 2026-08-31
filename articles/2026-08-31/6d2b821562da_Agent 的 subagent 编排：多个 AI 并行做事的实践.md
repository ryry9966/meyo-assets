---
title: Agent 的 subagent 编排：多个 AI 并行做事的实践
feedId: 35576
source: 综合讨论
publishedAt: 2026-08-31
---

## 背景
单个 Agent 处理复杂任务时容易遇到三类问题：上下文膨胀、工具权限混杂、串行等待。比如要同时调研三个仓库、审查五个接口、生成两份报告，如果都塞进主 Agent，中间结果会互相污染，一个节点失败还可能带崩整条链路。

subagent 编排的思路是把任务拆成多个独立 worker，主 Agent 只负责分发、校验和汇总。OpenClaw 里可以通过子任务/Agent 工具实现，配合 MCP 提供统一资源访问。

## 什么任务适合并行
适合：无强依赖、共享只读资源、输出可结构化合并。

不适合：强顺序依赖、需要实时协商、共享可变状态。

判断标准很简单：如果拆开后每个 worker 的输入输出边界清晰，且不需要中途互相等，就可以并行。

## 做法
1. 任务契约  
给每个 subagent 一个明确 schema，例如：
`{"goal": "...", "tools": ["read_github"], "output": "JSON only", "max_steps": 8}`  
输出要求只返回 JSON/Markdown，不输出解释。主 Agent 写任务时明确“不要补充建议、不要改格式”。

2. 上下文隔离  
每个 subagent 只注入任务相关材料，不全量灌文档。工具权限最小化：读 GitHub 的只给 read，写文件的只给指定目录。共享 MCP server 时用独立 session，避免 rate limit 互相影响。

3. 并行执行  
主 Agent 一次性 dispatch 3-5 个 subagent，设置 timeout 60-120s、max_steps 限制。异步等待结果，不阻塞交互。OpenClaw 的 agent 工具可以做并发，但每个 subagent 的任务 prompt 要独立，不要引用主对话里的隐含信息。

4. 结果校验  
主 Agent 按 schema 校验字段完整性。缺字段或 JSON 解析失败时，只对失败节点重试一次，重试 prompt 里加“previous output invalid, return only JSON”。

5. 合并与写操作收敛  
读和分析放在 subagent，写操作尽量集中到主 Agent 最后执行。若 subagent 必须写，要求幂等：创建前先检查是否存在，写入用临时文件再原子替换。

## 踩坑点
- 并行数不要贪多：3-5 个比较稳，过多会放大 token 消耗和输出方差。
- MCP 工具冲突：多个 subagent 共用同一 server 时可能出现限流或认证冲突，建议每 worker 独立 session 或加队列。
- 输出格式漂移：模型会漏字段、加解释文字，主 Agent 不要直接信任，用 JSON schema 校验。
- 隐蔽串行：汇总阶段主 Agent 读取所有结果，如果结果很长，主上下文又会爆。让 subagent 输出摘要，详细结果落到文件或队列。
- 循环依赖：用 DAG 拆任务，只做一层并行，避免 subagent 再嵌套 subagent 过深。

## 可复用建议
- 建模板：`researcher`、`reviewer`、`coder`、`formatter` 四个基础角色，按权限和输出 schema 复用。
- 主 Agent 用状态机：plan -> dispatch -> collect -> validate -> merge -> report，失败节点单独重试。
- MCP 统一资源访问：subagent 不直连 API，减少密钥散落。
- 先跑 2 个并行验证契约，再放大规模。

## 总结
subagent 编排本质是并发工程：任务有边界、契约清晰、失败可重试、写操作收敛。OpenClaw 适合做分发，但稳定性来自约束，而不是 prompt 技巧。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/9d82b484eb2438e1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/3e3c5343e6393a6a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-31/b05804e5c9bfbc4a.png)

