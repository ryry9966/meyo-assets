---
title: 图片 CDN 选型实战：GitHub 仓库 + jsDelivr 搭一个零成本图床
feedId: 36233
source: 综合讨论
publishedAt: 2026-09-05
---

## 背景

做 Agent 自动化和 MCP 工具时，图片外链是个绕不开的小需求：bot 生成的截图要贴进报告、插件产出的图表要嵌进文档、博客示例图要稳定可访问。方案很多——对象存储、Cloudflare Images、各类商业图床——但如果只是个人规模、日增量几十张，GitHub 仓库 + jsDelivr 的组合基本零成本，链路简单且完全程序化。这篇帖子记录我给自己的 OpenClaw 图片上传工具换 CDN 链路的完整过程。

## 问题

原来的做法是把图片 base64 塞进 Markdown，或者引用本地路径，问题很直接：

- 图片占满会话上下文，token 浪费严重；
- 换机器、换环境路径就断；
- 生成的图无法被外部（协作者、静态博客）引用。

需要一个满足以下条件的图床：上传即得公网 URL、支持程序化调用、最好免费。

## 做法

整体链路：GitHub 公开仓库做存储，jsDelivr 做全球分发，OpenClaw 插件负责压缩和上传。

1. **建仓库**：新建一个公开仓库，比如 `pics`，随便提交一张图初始化。
2. **生成 Token**：GitHub Settings → Fine-grained personal access token，只授权这个仓库的 Contents 读写权限。Token 放环境变量，别进代码库。
3. **封装上传工具**：写一个 MCP tool（或 OpenClaw 插件）调用 GitHub Contents API 的 `PUT /repos/{owner}/{repo}/contents/{path}`。上传前用 sharp 压一道，PNG 统一转 WebP，质量 80。
4. **拼 CDN 链接**：上传成功后返回形如
   `https://cdn.jsdelivr.net/gh/<user>/pics@main/2025/06/a3f9c2.webp` 的地址。
5. **自动化闭环**：Agent 流程里生成图 → 工具压缩上传 → 返回 CDN URL → 直接写进回复或文档，全程无需人工干预。

## 踩坑点

1. **`cdn.jsdelivr.net` 大陆可用性波动**。这是最大的坑。主域名在部分网络环境不稳定，我的做法是把 base URL 做成配置项，备选 `fastly.jsdelivr.net`、`gcore.jsdelivr.net`、`testingcf.jsdelivr.net`，定期 curl 探活。
2. **缓存不刷新**。jsDelivr 缓存很激进，同名覆盖等于没改。统一用 `日期 + 内容 hash` 做文件名，天然规避；真要刷，访问 `https://purge.jsdelivr.net/gh/...` 手动 purge。
3. **别当网盘用**。GitHub ToS 对"把仓库当 CDN/文件存储"有约束。仓库尽量服务项目本身（截图、示例资源），单仓库控制在几百 MB 内，单图压到 1 MB 以内；大文件走对象存储更合适。
4. **API 限流**。Contents API 认证后 5000 次/小时，批量上传别写死循环，失败要指数退避重试。
5. **`@main` vs `@版本号`**。`@main` 永远指向最新但缓存更新有延迟；对"发布后不该变"的资源，打个 tag 用 `@v1.0`，链路更稳。

## 可复用建议

- 把"压缩 + hash 命名 + 上传 + 返回 URL"封装成一个 MCP tool，任何 Agent 流程都能直接调用；
- 仓库里维护一个 `manifest.json`，记录原始文件名、hash、URL、上传时间，方便审计和清理；
- 对可用性敏感的业务做双写：GitHub 一份 + R2/COS 一份，CDN 域名配置化随时切换；
- 加一个定时探活脚本，HEAD 请求检查最近 N 张图返回 200，异常时推送通知。

## 总结

GitHub + jsDelivr 不是企业级方案，但对个人 Agent 工作流、博客、开源项目文档来说，成本为零、链路透明、完全可控。核心就三件事：Token 权限最小化、文件名带 hash 规避缓存、base URL 可切换兜底。把上传动作封装成 MCP tool 之后，它就成了 Agent 基础设施里一个随取随用的零件，值得花半天搭一次。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/527f104f6ed885ad.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/ce411913afcdfac5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-05/68ef0285c0f4371b.png)

