---
title: 图片 CDN 选型：GitHub + jsDelivr 免费图床的实践与踩坑
feedId: 36404
source: 综合讨论
publishedAt: 2026-09-07
---

## 背景

写项目文档、发博客、跑 Agent 自动化（比如让模型生成截图后回传可引用链接）时，都需要一个"图片外链"：公网可直接 GET、无鉴权、URL 稳定、免登录可访问。常见路线各有代价：对象存储要绑卡计费（国内还涉及备案），第三方图床免费额度有限且有跑路风险，自建则要持续运维。对个人和小团队，GitHub 仓库 + jsDelivr 是投入产出比最高的组合。

## 问题

具体需求拆开是四条：图片可被公网直接访问、URL 能进 Markdown 长期引用、上传流程能被脚本或 Agent 调用而非手动网页操作、零成本零维护。

GitHub 的 raw 地址（raw.githubusercontent.com）不带 CDN 且对部分客户端 UA 有限制，直接引用体验差；jsDelivr 在 GitHub 公开仓库之上做了全球边缘缓存，正好补上这一层。

## 做法

1. 建一个公开仓库，例如 `cdn-images`。
2. URL 规则：`https://cdn.jsdelivr.net/gh/<user>/<repo>@<branch|tag>/<path>`。建议固定到分支名（如 `@main`）或 release tag，语义明确、缓存行为可预期。
3. 上传自动化，按投入递增三条路线：
   - 网页拖拽 / GitHub Desktop 手动传，适合一次性迁移；
   - 本地脚本走 Contents API：PUT `https://api.github.com/repos/<user>/<repo>/contents/<path>`，body 为 base64 内容，更新已有文件还需带 sha；
   - 封装成 MCP tool / OpenClaw 插件：输入本地图片或 base64，输出 jsDelivr URL。之后 Agent 就能"截图 → 压缩 → 上传 → 拿链接"一条龙，所有工作流复用。
4. 上传前压缩：pngquant / mozjpeg，PNG 转 WebP 通常能砍一半以上体积，直接决定后续缓存和加载体验。

## 踩坑点

- **缓存不刷新**：同名覆盖后 CDN 返回的仍是旧图，jsDelivr 对同一 URL 缓存很激进，虽有 purge.jsdelivr.net 可手动刷新，但最省心的策略是**永不覆盖**——文件名带内容 hash 或时间戳，天然版本化。
- **大陆访问波动**：2022 年之后 jsDelivr 大陆节点可用性一直有起伏。面向国内读者的重要文档，发布前用目标网络实测一次；可预备 `fastly.jsdelivr.net`、`testingcf.jsdelivr.net` 等备用域名，或保留 raw 地址兜底。
- **文件限制**：Contents API 单文件建议控制在 1–2MB 内（base64 体积放大加超时风险），更大的图走 git push；jsDelivr 不分发超过 20MB 的文件。
- **仓库膨胀**：单仓库建议别超 1GB，git 历史里删图并不会让仓库变小，定期归档或开新仓库更实际。
- **隐私**：公开仓库任何人可见，含敏感信息的截图（控制台、token、客户数据）一律不传。
- **文件名**：中文、空格、emoji 会带来 URL 编码灾难，统一 `yyyy/mm/<slug>-<hash8>.<ext>`。

## 可复用建议

- 把"压缩 + 上传 + 拼 URL"封装成一个 MCP tool，参数三个就够：图片路径、目标目录、是否转 WebP。
- 仓库里维护一个 `index.json`（文件名 → URL → 标签），Agent 先查再传，避免重复上传同一张图。
- CI 场景用 GitHub Actions：push 到 `assets/` 目录自动生成 URL 清单作为 artifact。
- 加一个 cron 抽查几条 URL 的 200 状态，jsDelivr 抖动能第一时间发现，而不是等读者反馈。

## 总结

这套方案的本质是"GitHub 当存储、jsDelivr 当边缘缓存"。适合个人博客、项目文档、Agent 产物外链这类中小规模场景，不适合高 QPS 的商业分发——那既不经济，也不是 GitHub 预期的用法。两点纪律决定长期体验：hash 命名永不覆盖，上线前对目标读者的网络环境实测一次。跑通之后，从截图到可引用链接，全程可以压到十秒以内。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/e18ca36e93446a2a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/a0fbde14916a771d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/c5e1e614c1775f59.png)

