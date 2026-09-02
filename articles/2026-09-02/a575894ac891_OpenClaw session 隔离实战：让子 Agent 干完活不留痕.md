---
title: OpenClaw session 隔离实战：让子 Agent 干完活不留痕
feedId: 35857
source: 综合讨论
publishedAt: 2026-09-02
---

## 背景

OpenClaw 的主会话承载两样东西：用户可见的对话历史，以及 agent 的工具调用轨迹——hooks、cron、插件消息都会往里写。一旦开始用 spawn 派子 agent 跑检索、批处理、长任务，主会话很快就会被中间过程撑爆：compact 提前触发，摘要里混进一堆无关的工具日志，之后每一轮对话都在为这些垃圾付 token。

## 我们遇到的问题

早期图省事，子任务直接在主会话里跑，典型症状：

- 主会话上下文从 3k 涨到 30k，回答质量肉眼可见地变差；
- 子任务的失败重试记录永久留在历史里，模型后续行为被带偏；
- 两个 cron 任务并发往主会话写，出现交错消息和误触发。

## 做法

1. **子任务一律走独立 session。** spawn 时给每个子 agent 分配独立的 session key 和 workspace，不共享主会话文件。我们按 `${task_id}` 做目录约定，物理隔离，出了问题直接翻对应文件。
2. **上下文做减法。** 给子 agent 的 prompt 只包含：任务目标、输入数据路径、输出格式要求，绝不把主会话历史粘进去。
3. **回传走摘要通道。** 子 agent 结束后只回传结构化结果（状态 + 关键数字 + 产物路径），过程日志留在子会话里按需查看。
4. **权限收敛。** 子 agent 的工具白名单按任务给，写权限只落自己的 workspace，主目录只读。
5. **生命周期管理。** 子会话带 TTL，结束后归档，过期文件由定时脚本清理。

配置示意：

```yaml
subagent:
  session: "sub-${task_id}"     # 独立 session key
  workspace: "work/${task_id}"  # 独立工作目录
  inherit_context: false        # 不继承主会话历史
  result: summary               # 回传只取摘要
  max_result_lines: 40
  ttl_days: 7
```

## 踩坑点

- **忘了关 `inherit_context`**：子 agent 把主会话的调试历史当成任务背景，行为完全跑偏。改完配置要用新任务验证，别拿旧会话测。
- **回传结果没限长度**：子 agent 把整份 JSON 扔回主会话，照样被撑爆——隔离了个寂寞。摘要通道必须强制限行。
- **并发写共享文件**：多个子 agent 写同一个产物路径互相覆盖。改成各写各的目录，由主 agent 汇总。
- **工具白名单漏裁剪**：子 agent 沿用了主 agent 的完整工具清单，一次纯检索任务顺手调了写入工具。

## 可复用建议

- 任务描述做成模板：目标 / 输入 / 输出格式 / 禁止事项，四段式，稳定且省 token；
- 结果摘要定义成固定 schema，主 agent 的后续推理只依赖 schema 字段；
- 每周看一次 agents 目录下子会话文件大小，异常膨胀说明有任务在漏日志；
- 排查问题时先看子会话原文，再回主会话对照，顺序别反。

## 总结

session 隔离的本质不是“多开几个文件”，而是控制信息流向：任务下去时做减法，结果回来时做压缩，权限按任务最小化。这套做完后，我们主会话上下文稳定在 5k 以内，compact 基本不再被中间过程触发。该脏的地方让子 agent 脏在自己的会话里，主会话只留干净的结果。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/97e16d691e73726d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/82fce96b08c1bbe8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/f77208b6de5c5b6c.png)

