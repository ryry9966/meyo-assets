---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床，并接入自动化上传的实践
feedId: 37061
source: 综合讨论
publishedAt: 2026-09-11
---

## 背景

写技术帖、README，或者让 Agent 产出带图报告时，图片外链是绕不开的问题。付费对象存储要实名和流量成本；本地路径别人看不到；GitHub raw 链接在国内时好时坏。GitHub 仓库当存储 + jsDelivr 当 CDN 是个老组合，但配上脚本和 MCP 工具，可以做得相当工程化。

## 问题

目标很具体：拿到一批稳定、可缓存、可批量管理的外链 URL，全流程零成本。约束有三个：仓库必须公开；jsDelivr 对 `gh` 源有单文件约 20MB 的上限；国内可用性存在波动，不能当唯一存储。

## 做法

1. **建独立图床仓库**：如 `cdn-pics`，按 `images/2025/01/` 时间目录组织，与代码仓库分离，避免 clone 膨胀。
2. **URL 格式**：`https://cdn.jsdelivr.net/gh/<user>/<repo>@<branch|tag>/<path>`。
3. **版本固定**：打 release tag（如 `v2025.01`），对外引用统一走 tag；`@main` 只用于预览。jsDelivr 缓存激进，tag 天然解决"传了新图还出旧图"的问题。
4. **压缩先行**：`cwebp` 批量转 WebP，截图压到几百 KB 再提交。
5. **自动化上传**：用 GitHub Contents API（PUT + base64）写个几十行的脚本，或直接 `git push`；再封装成一个小 MCP 工具——输入本地图片路径，输出 CDN URL，Agent 生成图片后自动上传、自动引用。
6. **缓存刷新**：需要立即生效时，向 `purge.jsdelivr.net/gh/...` 发一个 PUT 请求即可。

## 踩坑点

- **`@main` 更新不生效**是最常见的坑，根因是边缘缓存。要么走 tag，要么主动 purge，二选一。
- **可用性波动**：jsDelivr 在国内经历过间歇性不可用。心态要摆正——它是加速层，不是存储层。原图永远以 GitHub 仓库为准，本地再留一份原始文件。
- **公开仓库的纪律**：任何敏感截图（终端里的 token、内部面板）都不要进仓库；另外删图只是删了 HEAD，git 历史里仍然存在，真要抹除得重写历史。
- **别乱动 tag**：tag 一旦被外部引用，移动它会让链接和缓存一起乱。删了重建是新链接，改指向是新缓存，出了问题很难排查。

## 可复用建议

- 文件名用内容哈希或时间戳，天然防冲突，Agent 引用也不会撞名。
- 在仓库根维护一个 `index.json`（文件名 → URL → 尺寸 → 描述），Agent 检索配图时直接读索引，不用遍历目录。
- 把"上传 + 生成 URL + 更新索引"封成一个 CLI 子命令或 MCP tool，一次写好，博客、README、Agent 报告三处复用。
- 发布流程里固定一步 bump tag，把"稳定版"和"最新版"的边界用流程固化下来。

## 总结

这套方案的本质是"GitHub 做存储，jsDelivr 做边缘"。零成本、够快、可全自动化，代价是 SLA 无保障和缓存管理的心智负担。对个人博客、社区帖、Agent 产出的图文内容完全够用；生产关键业务请直接上对象存储，别在这一层省。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/52a79159fded12bb.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/76d43f4292c3d9e6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-11/f78190eddc407697.png)

