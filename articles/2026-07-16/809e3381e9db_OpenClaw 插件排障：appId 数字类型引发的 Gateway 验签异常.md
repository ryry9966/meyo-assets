---
title: OpenClaw 插件排障：appId 数字类型引发的 Gateway 验签异常
feedId: 29320
source: Bug反馈
publishedAt: 2026-07-16
---

## 背景

团队最近在 OpenClaw 上封装一个内部插件，用于调用公司统一 API 网关。网关要求调用方提供 `appId` 与 `appSecret`，并基于它们生成签名。所有接入方都已经按这套规范跑了大半年，结果唯独在 OpenClaw 插件上一直返回 `SignatureNotMatch`，让人一度怀疑是框架本身对签名算法做了某种魔改。

## 问题

插件开发完成后，第一次联调就直接撞上 401。网关返回的 `X-Ca-Error-Message` 写得很清楚：`Signature does not match`。通常情况下，这是签名字符串拼接顺序错误，或者多了一个换行符。但这次我们反复比对了文档与代码，签名逻辑看不出任何问题。

于是回到日志，打印出插件配置对象：

```json
{
  "appId": 1.2345678901234568e+19,
  "appSecret": "abc123"
}
```

我们配置的 `appId` 本应是 `12345678901234567890`——一个标准的 20 位数字型 ID。但在 OpenClaw 的运行时里，它已经变成了科学计数法表示，且末尾精度已经丢失。当这个值被拼接到签名原文中，计算出的摘要自然与网关预期截然不同。

## 做法 / 步骤

**1. 确认变量实际值**

在插件的 `execute` 钩子里加入一行临时日志，输出 `JSON.stringify(this.config)`。这一步直接暴露出 `appId` 已经被解析成 `number`，并且由于值远超 `Number.MAX_SAFE_INTEGER`（2⁵³−1，约为 9e15），JSON 序列化后精度丢失。

**2. 回溯配置来源**

插件配置最初写在 `plugin.yml` 中：

```yaml
config:
  appId: 12345678901234567890
  appSecret: "abc123"
```

YAML 解析器在没有引号的情况下，会把 `12345678901234567890` 当成整数处理。尽管 OpenClaw 底层使用 JavaScript，但 YAML 解析库会将这个长数字转换为 `number`，导致精度丢失发生在配置加载阶段，远早于签名计算。

**3. 修复方式**

最简单的修复是为 `appId` 加上引号，使 YAML 将其解释为字符串：

```yaml
config:
  appId: "12345678901234567890"
  appSecret: "abc123"
```

重新部署后，日志中的 `config.appId` 变成 `"12345678901234567890"`，签名通过，网关返回正常业务数据。

**4. 插件配置 Schema 加固**

为了防止团队其他人再次栽进同一个坑，我们在 OpenClaw 插件的 `schema.json` 里把 `appId` 字段明确声明为 `string`：

```json
{
  "appId": {
    "type": "string",
    "description": "网关分配的 appId，必须为字符串"
  }
}
```

这样即便有人在 YAML 中错误写成数字，框架也会根据 schema 进行类型校验并给出提示，而不是静默地引入精度问题。

## 踩坑点

- **YAML 数字陷阱**：长数字 ID 在 YAML 中不加引号就会被解析为 `int`/`float`。银行账号、外部系统 ID、乃至手机号，都有机会踩这个坑。
- **JavaScript 安全整数边界**：任何超过 9007199254740991 的整数值在 JavaScript 中都可能失精。我们这次用的 20 位 appId 刚好超出，表现为末尾若干位变成 0。
- **JSON 序列化无声丢失**：`JSON.stringify` 对大整数也会丢失精度，并且不会抛出异常，排查时容易被日志本身误导——你可能看到了科学计数法，却误以为只是显示问题。
- **网关签名对类型敏感**：即使有些网关 SDK 会在内部将数值型 appId 转为字符串，签名计算时拼接字符串往往仍然依赖于传入的原始类型。数字 `123...` 与字符串 `"123..."` 在拼接时的表现并不相同，不要寄希望于底层自动修正。

## 可复用建议

1. **永远用字符串承载 ID、Token 及长数字**。不管是 YAML 配置、环境变量还是数据库字段，只要不参与数值运算，就用字符串。
2. **在插件 JSON Schema 中显式约束类型**。OpenClaw 的插件体系允许自定义 schema，利用 `type` 字段可以提前拦截不合理的数字输入。
3. **网关对接第一步先抓包看 “实际发出的原文”**。比对手写签名与请求中携带的签名，通常会更快定位是拼接原文问题还是密钥问题。
4. **团队内部建一份「配置红线」**：列出所有禁止写成数字的字段（ID、token、secret、phone 等），并结合 ESLint 或 Schema 校验落地。

## 总结

这次排障表面上看是 YAML 少写两个引号，本质上是跨技术栈的类型不一致被网关签名机制放大。OpenClaw 作为一个高度依赖配置的插件化平台，开发者也需要对 YAML → JavaScript 的类型转换保持警惕。少一点“顺手”，多一分 Schema 约束，往往就能让网关 401 这类问题止步于联调之前。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/777c4d65fc3b4e79.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/d2cdf9106a03d44c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-07-16/468fc49eea251c50.png)

