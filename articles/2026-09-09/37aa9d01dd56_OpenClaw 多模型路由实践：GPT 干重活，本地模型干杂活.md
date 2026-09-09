---
title: OpenClaw 多模型路由实践：GPT 干重活，本地模型干杂活
feedId: 36762
source: 综合讨论
publishedAt: 2026-09-09
---

## 背景

OpenClaw 里的 Agent 任务其实并不均匀。一部分是“重活”：多步工具调用、代码生成、长上下文推理；另一部分是“杂活”：清洗输出、生成摘要、格式转换、批量打标签。如果全部走 GPT，账单和延迟都难看；如果全部走本地模型，复杂任务的工具调用成功率和稳定性会明显下滑。多模型路由解决的就是这个分配问题。

## 问题：按“能力”切，而不是按“品牌”切

早期我们的规则很粗：默认 GPT，本地模型只做兜底。跑下来发现约六成请求是本地模型完全能胜任的轻任务，云端配额被无意义消耗。反过来也有反例：把 MCP 工具调用交给 7B 本地模型后，函数参数幻觉率明显上升。结论是路由维度应该是任务类型、上下文长度、敏感级别，而不是“哪个模型名气大”。

## 做法：三层路由策略

我们最终收敛成三层规则：

1. **能力层**：带 tool_call 且步骤数大于 2 的任务路由到云端强模型；纯文本摘要、改写、分类路由到本地模型。
2. **长度层**：上下文超过本地模型窗口的 80% 直接走云端，避免截断导致的静默失败。
3. **敏感层**：标记了 private 的会话（内部代码、客户数据）强制本地，宁慢不外发。

OpenClaw 的 router 配置大致如下：

```yaml
routes:
  - match: { tools: true, steps_gt: 2 }
    target: cloud-strong
  - match: { ctx_tokens_gt: 51200 }
    target: cloud-strong
  - match: { privacy: private }
    target: local-only
  - default: local-fast
fallback: [cloud-strong, local-fast]
```

fallback 链很关键：本地服务挂了或超时，自动切云端，Agent 流程不中断。

## 踩坑点

- **工具调用 schema 不兼容**：本地模型输出的函数参数格式和 GPT 不完全一致，插件里的参数校验要按模型分别调，别假设一份 prompt 通吃。
- **上下文窗口是硬约束**：小窗口本地模型接长文档，截断不报错，只会安静地给出错误答案，比直接失败更难排查。
- **延迟要测端到端**：本地模型首 token 快但吞吐慢，长输出场景总耗时可能反而超过云端 API。
- **路由日志必须落盘**：不记录 route decision，出了质量问题根本无法回溯责任在哪个模型。

## 可复用建议

- 把路由策略当代码管理，进版本库，改规则要有 commit 记录。
- 每条路由配一个小评测集（20~50 条样例），换模型或改 prompt 后跑一遍，防止质量漂移。
- 本地模型用 Ollama/vLLM 常驻服务，别每次冷启动，否则“省钱”变成“费电费时间”。
- 先按 70/30（本地/云端）的目标切流量，再根据评测结果微调，不要一步到位。

## 总结

多模型路由不是炫技，是成本、质量、隐私三者之间的工程折中。原则很简单：重活给云端强模型，杂活和敏感活给本地模型，中间用 fallback 和日志兜底。先把任务分类做对，路由规则自然会简单下来。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/e00b2c0c47360daa.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/7b9ba964d32493a6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/789e94b88d35f78d.png)

