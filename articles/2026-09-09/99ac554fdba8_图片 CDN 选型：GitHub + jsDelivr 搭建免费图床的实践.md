---
title: 图片 CDN 选型：GitHub + jsDelivr 搭建免费图床的实践
feedId: 36730
source: 综合讨论
publishedAt: 2026-09-09
---

## 背景

写技术帖、做 agent 内容产出的过程中，截图和插画是刚需。OpenClaw 的自动化流程里，agent 生成的文章、评测、知识卡都需要稳定的图链。图床的选型大致三条路：对象存储（要钱，还要绑域名）、免费图床服务（外链不稳定，随时可能跑路）、GitHub 仓库 + jsDelivr（免费、版本化，国内访问有运气成分）。

我最终选了第三条。理由很简单：图片本身就该进版本管理，而且整条链路可以被脚本和 MCP 工具完全自动化。

## 做法

核心三步：

1. 建一个公开仓库，比如 `assets`，专门放图。
2. 推送后通过 jsDelivr 的 GitHub 源访问：

```
https://cdn.jsdelivr.net/gh/<user>/<repo>@<tag>/<path>
```

3. 把上传动作包成小工具，接进 agent 工作流。

自动化部分我写的方案是一个 MCP tool：输入本地图片路径，内部做两件事——压缩（pngquant），然后调 GitHub Contents API 上传，返回拼好的 jsDelivr URL。这样 agent 生成文章时可以直接引用，不用人肉搬图。Token 用 fine-grained PAT，只授予这一个仓库的 Contents 读写权限，泄露影响面可控。

## 踩坑点

- **缓存不刷新是最大的坑。** 用 `@main` 引用时，同名文件覆盖后 jsDelivr 会长期返回旧版本。两个解法：文件名带内容 hash 天然规避；或上传后手动 purge（`purge.jsdelivr.net/gh/...`）。我选前者，顺带解决了命名冲突。
- **不要用中文文件名。** URL 编码问题会让部分客户端拿到 404，统一用 `YYYYMM-<hash>.png`。
- **jsDelivr 不是给图床滥用用的。** 它的定位是开源项目 CDN，单文件超过 20MB 不提供服务，仓库体积无限膨胀也有被限流的风险。我控制单图 500KB 以内、仓库总量几百 MB，目前没出过问题。
- **国内可用性要留后手。** `cdn.jsdelivr.net` 这几年时好时坏，可准备 `fastly.jsdelivr.net` 备用域名，发布前两个都验证一遍。
- **公开仓库别放隐私截图。** agent 生成的日志截图尤其注意，上传前先脱敏，这是流程问题不是技术问题。

## 可复用建议

- 命名规则固定为 `日期-内容hash.扩展名`：幂等且缓存友好，重复上传同一张图可直接短路返回旧 URL。
- MCP tool 里加一步校验：上传完对 CDN URL 发一次 HEAD 请求，确认 200 再把链接交给 agent，避免下游拿到失效链接。
- 图更新频繁的场景改用 `@<commit-sha>` 引用，精确版本与长缓存两全。
- 压缩这步别省。存储成本为零，但加载速度是读者的真实体验。

## 总结

GitHub + jsDelivr 适合个人博客、技术文档、agent 产出内容这类中小流量场景，核心收益是免费、版本化，以及整条链路可以塞进自动化流水线。它不适合当生产级 CDN 用——没有 SLA，随时可能变脸。在 OpenClaw 实践里我的体会是：选型的关键不只是"图放哪"，而是"图链怎么被 agent 稳定拿到"。把压缩、上传、校验封成一个 MCP 工具之后，这件事才算真正解决。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/17528b70985eff7c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/48711cbea77aaabd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/802d5db8980a4d61.png)

