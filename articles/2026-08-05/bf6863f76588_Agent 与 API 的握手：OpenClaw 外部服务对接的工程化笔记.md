---
title: Agent 与 API 的握手：OpenClaw 外部服务对接的工程化笔记
feedId: 31704
source: 综合讨论
publishedAt: 2026-08-05
---

## 背景

OpenClaw 的角色是 agent 壳：守着你指定的通道（Telegram、Slack 或本地 CLI），把用户意图转成一次工具调用。而工具的本质，就是外部 API 的代理。社区里最常见的求助帖不是"怎么写 prompt"，而是"为什么我的 agent 调不通外部服务"。

## 问题

LLM 是决策者，API 是执行者，中间胶水层的成败由三件事决定：工具如何注册、鉴权如何管理、错误如何返回。

很多项目死在第二和第三件事上：轮询 token 过期没人管，外部服务 5 秒超时，agent 就反复重试同一个请求，最后给你一句"服务可能暂时不可用"——这是典型的错误信息被 LLM 脑补的结果。

## 做法

对接前先做决策：

- 标准 HTTP API 且有复用诉求 → 用 MCP Server 包一层；
- 一次性脚本、内部小工具 → 用 OpenClaw 扩展即可；
- 高频调用、有重试和缓存诉求 → 自建 adapter。

以最简 OpenClaw 扩展为例，对接 GitHub Issue API：

```js
module.exports = {
  tools: [{
    name: "get_issue",
    description: "Get a GitHub issue by owner/repo/number",
    parameters: {
      owner: "string",
      repo: "string",
      number: "number"
    }
  }],
  async execute(name, args) {
    const url = `https://api.github.com/repos/${args.owner}/${args.repo}/issues/${args.number}`;
    const res = await fetch(url);
    return { ok: res.ok, status: res.status, body: await res.text() };
  }
};
```

要点：把 HTTP status 显式返回，并保留 body 摘要。LLM 需要知道"调用到达了对端但没拿到 200"，否则它会在错误信息上过度推理。

若走 MCP，写一个 30 行的 stdio server 即可，注意工具描述里注明触发条件。

## 踩坑点

1. **MCP over stdio 的日志污染。** 很多人习惯在 server 里 `console.log` 调试，直接破坏协议流。日志必须走 stderr 或独立文件。
2. **同名工具静默覆盖。** 多个扩展注册了同名的 `get_issue`，后加载的覆盖先加载的，agent 行为诡异且无任何报错。建议启动时打印工具注册表，肉眼核对一遍。
3. **超时和重试语义不清。** OpenClaw 默认工具超时偏短，外部服务偶发变慢时，agent 会误判为"工具不存在"而不是"服务慢"。在工具层做好 `timeout + retry`，把不确定性消化在内部。
4. **Token 刷新没人管。** 401 响应直接丢给 LLM，它会试图自己"修复"：换参数、加 header、反复试。正确做法是在工具层拦截 401，统一返回"凭证失效，需要人工重新配置"。

## 可复用建议

- 工具参数用 JSON Schema 严格定义，缺字段就报参数错误，不要让 LLM 用空值瞎调。
- 写 POST 类工具时自带 idempotency key，防止 agent 重试导致重复下单/重复建单。
- `description` 里写清楚"何时用、何时不用"，这是唯一能有效抑制乱调用的杠杆。
- 对接前先用 curl 手工验证 API，再写扩展，最后才接 agent。隔离变量，排障速度快一个量级。

## 总结

OpenClaw 对接外部服务没有魔法，代码量也不大。真正的复杂度在错误语义和生命周期管理。把所有不确定性隔离在工具层，让 agent 拿到的永远是干净、可解释的结果——这套系统才谈得上可用。

---

