---
title: OpenClaw 定时任务选型：cron 与 heartbeat 的分工与踩坑
feedId: 37067
source: 综合讨论
publishedAt: 2026-09-11
---

## 背景

OpenClaw 里的 agent 有两条被唤醒的路径。一条是 **cron**：用 cron 表达式在指定时间触发任务，到点必跑；另一条是 **heartbeat**：agent 按固定间隔醒来，读一遍工作区的 `HEARTBEAT.md`，自己判断这次要不要做事。两者本质上都在消耗 token，但语义完全不同。

## 问题

见过两类典型误用：

1. **把 heartbeat 当 cron 用**——在 `HEARTBEAT.md` 里写“每天早上 9 点发日报”，结果触发时间漂移、机器休眠时直接漏发；
2. **用 cron 做巡检**——每小时跑一次“检查有没有异常”，绝大多数触发是一次昂贵的“没事”，token 消耗曲线很难看。

一句话概括差异：cron 是“到点必跑”，heartbeat 是“醒来看一眼，自己决定跑不跑”。

## 做法

我的选择标准：

- 时间敏感、必须准点、动作固定 → **cron**
- 状态巡检、有事才说、允许 agent 自行判断 → **heartbeat**
- 纯被动响应消息 → 两个都不用开

heartbeat 配置（`openclaw.json`）：

```json
{
  "agents": {
    "defaults": {
      "heartbeat": { "every": "45m" }
    }
  }
}
```

巡检项写进 `HEARTBEAT.md`，务必写成可判定的条件，并显式声明“无事则沉默”：

```markdown
- 检查 ~/reports 下是否有昨天生成失败的文件；有则报告，无则忽略。
```

cron 用 CLI 建 job（参数以 `openclaw cron add --help` 为准）：

```bash
openclaw cron add --name morning-digest \
  --cron "0 9 * * *" \
  --message "汇总昨天的邮件与日历要点" \
  --deliver --channel telegram
```

## 踩坑点

1. **heartbeat 间隔不是精确承诺**：网关重启、机器休眠期间不会触发，恢复后也不补跑。别拿它做准点提醒。
2. **HEARTBEAT.md 写模糊指令**（“看看有没有重要的事”）会导致 agent 频繁误报，或每一拍都回复，token 直接翻倍。
3. **cron 表达式按网关所在时区解析**，VPS 上默认可能是 UTC；必要时用 `--tz` 显式指定，建完先确认下次触发时间。
4. **cron 触发即烧 token**，与 payload 长短无关；每 5 分钟一次的巡检别用 cron，要么用 heartbeat 拉长间隔，要么改成事件驱动（webhook 推送）。
5. 心跳和 cron 往**同一个会话**里灌内容会让上下文膨胀，巡检类任务建议隔离 session 或定期清理。

## 可复用建议

- **一句话判断法**：能写成 cron 表达式的需求用 cron；只能描述成“每隔一段时间看看有没有 X”的用 heartbeat。
- heartbeat 间隔**宁长勿短**，45–60 分钟起步，跑一周看 token 消耗再调。
- 把 `HEARTBEAT.md` 当 checklist 维护：每条都是布尔条件，明确“不满足则沉默”。
- 真正需要分钟级持续监控的场景不要硬塞给 agent，用外部监控告警推 webhook 更省心。

## 总结

cron 和 heartbeat 不是竞争关系，而是两种唤醒语义。把“什么时候干”的确定性交给 cron，把“要不要干”的判断权交给 heartbeat，是我实际用下来最省心的分工。先用起来，再根据 token 账单微调。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/72f6e7c482b0593c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/537b49a50ce6e6b8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/698eda2cb1a837ab.png)

