---
title: 图片 CDN 选型实录：GitHub 仓库 + jsDelivr 搭免费图床的工程化实践
feedId: 36567
source: 综合讨论
publishedAt: 2026-09-08
---

## 背景

写博客、文档、Agent Demo 时经常需要图片外链。方案对比过几类：商业对象存储（要绑卡计费，图床场景杀鸡用牛刀）、公共免费图床（随时跑路）、GitHub raw 直链（国内直连很慢）。最后落到 **GitHub 仓库当存储 + jsDelivr 当 CDN** 这条路：零成本、URL 稳定、最关键是可以完全脚本化，能嵌进 Agent 的自动化流水线。

## 问题

三个具体需求：

1. Markdown 写作时贴图即得外链，编辑器里直接引用；
2. 静态博客加载图片速度要能接受；
3. Agent 自动产图（架构图、截图）后，能自己完成「上传 → 拿 URL → 写进文章」的闭环，不经过人手。

## 做法

1. 建一个 public 仓库，比如 `cdn-assets`，专门放图，和工作仓库隔离。
2. 图片推上去后，外链格式为：
   `https://cdn.jsdelivr.net/gh/<user>/<repo>@<branch>/<path>/img.png`
3. 自动化上传直接调 GitHub Contents API：`PUT /repos/{owner}/{repo}/contents/{path}`，body 里放 base64，一次请求等于一次 commit。封装成 20 行左右的脚本，MCP tool 或 agent 工具直接调用即可。
4. 压缩是必须步骤：用 sharp 或 pngquant 先压，单图控制在几百 KB。
5. 命名规则用 `YYYYMMDD-<内容hash>.<ext>`，天然幂等，也避免同名覆盖撞缓存。
6. 覆盖旧文件后如需立即生效，请求 `purge.jsdelivr.net` 对应路径强制刷新。

## 踩坑点

- **jsDelivr 大陆可用性有波动**，这几年时好时坏。缓解办法：域名可替换为 `fastly.jsdelivr.net`、`gcore.jsdelivr.net` 等镜像域，前端做多域名 fallback；重要场景别单点依赖。
- **单文件超过 20MB jsDelivr 不分发**，GitHub 硬限 100MB。图床场景本就不该出现这种文件，压缩前置。
- **用 `@branch` 引用时缓存有滞后**，覆盖同名文件后可能拿到旧图。实践中换文件名最省心，purge 接口偶尔抽风。
- Contents API 未认证限 60 次/小时，**务必带 token**，且 token 只授权这一个仓库。
- public 仓库意味着**任何猜到 URL 的人都能访问**，敏感截图别放。
- GitHub 对「把仓库当网盘」是有态度的，控制量级，只放博客/文档资产，别拿来分发安装包。

## 可复用建议

- 把上传脚本包成 MCP tool（入参：本地路径或 base64；出参：外链 URL），Agent 写文章的流水线即可自动插图。
- **仓库本身是 source of truth，jsDelivr 只是缓存层**。生成外链时把域名前缀抽成配置项，将来迁 R2/OSS 只改一处。
- 文件名带内容 hash，重复上传同一张图不产生脏数据，方便幂等重试。
- 本地留一份上传日志（时间、文件名、URL），审计和迁移时救命。

## 总结

这套方案的本质是「GitHub 当存储、jsDelivr 当缓存」：免费、够用、可脚本化，适合个人博客、文档和 Agent 自动化产图。但要认清它的定位——best-effort 基础设施，可用性和政策都有变数。工程上的正确姿势是把它做成**可替换的一层**：前缀可配置、hash 命名、日志留痕。真到要搬家那天，半小时能迁完，这就是选型时该留的后路。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/0e431265a6003ece.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/a3bdc0536694b911.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-08/a39937580f7a9a32.png)

