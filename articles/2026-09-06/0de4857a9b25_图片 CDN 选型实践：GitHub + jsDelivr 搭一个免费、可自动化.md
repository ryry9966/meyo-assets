---
title: 图片 CDN 选型实践：GitHub + jsDelivr 搭一个免费、可自动化的图床
feedId: 36356
source: 综合讨论
publishedAt: 2026-09-06
---

## 背景

写博客、维护文档、让 Agent 输出带截图的报告，都绕不开图片托管。可选路径无非几条：对象存储（要钱、要实名）、第三方免费图床（随时可能跑路或加防盗链）、GitHub raw 链接（大陆访问极不稳定）。jsDelivr 在 GitHub 仓库之上套了一层全球 CDN，等于把「用 Git 管理的图片仓库」变成了一个免运维的图片源，这是目前个人开发者成本最低的方案之一。

## 遇到的问题

- `raw.githubusercontent.com` 在大陆基本不可用，README 和博客里的图经常裂；
- 免费图床链接寿命不可控，迁移成本高；
- 自动化场景需要 URL 规则稳定可预测——我经常让 Agent 截图后自动贴进 issue 和笔记，链接必须是程序拼出来的，不是手工传的。

## 做法

1. **建一个公开仓库**，比如 `assets`，按 `项目/年份/文件名` 组织目录，别全堆在根目录。
2. **引用格式**用 jsDelivr 的 gh 源：
   `https://cdn.jsdelivr.net/gh/<user>/<repo>@<commit-sha>/blog/2025/demo.webp`
   生产环境建议钉 commit SHA 而不是 `@main`，URL 天然不可变。
3. **自动化上传**：写了个小脚本（Python + GitHub Contents API），流程是：剪贴板取图 → 压缩转 WebP → 文件名加内容 hash → PUT 上传 → 拼出 CDN URL 返回。在 OpenClaw 里包成一个 skill/MCP 工具后，Agent 可以直接说「把这张截图传上去给我链接」，全程无需人肉操作。
4. **覆盖更新后清缓存**：访问 `https://purge.jsdelivr.net/gh/<user>/<repo>@<branch>/<path>` 可主动刷掉旧缓存。

## 踩坑点

- **单文件 20MB 上限**：jsDelivr 不服务超过 20MB 的 gh 文件，截图前务必压缩，PNG 转 WebP 通常能砍 70% 体积。
- **分支 URL 有约 12 小时缓存**：同名覆盖不会立即生效，要么用 hash 文件名只增不改，要么走 purge 接口。
- **仓库必须公开**：这意味着绝不能传敏感截图。给自动化工具加了一层敏感关键词检查，防止 Agent 把含 token、内网地址的图传上去。
- **大陆可用性有波动**：jsDelivr 过去出过解析故障，我在博客端做了 `onerror` 回退到 raw 链接的双保险。
- **注意 ToS 边界**：jsDelivr 定位是开源项目文件分发，当个人图床属于灰色用法。控制总量（仓库 < 1GB）、不传视频、不上量，一般没人管，但心里要有数。

## 可复用建议

- **文件名 = 内容 hash**：天然去重、缓存永久有效，同一张图重复上传幂等返回同一 URL，Agent 重试也安全。
- **仓库即备份**：图片和代码一样有完整版本历史，未来换 CDN 只需换 URL 前缀，迁移是纯字符串替换。
- **批量上传加退避**：GitHub API 有速率限制，脚本里 sleep 一下，别把 403 练成肌肉记忆。
- 回退源可以备一个 Statically，同样是基于 GitHub 仓库的 CDN。

## 总结

GitHub + jsDelivr 不是最稳的方案，但它免费、有版本管理、URL 规则透明，尤其适合「Agent 自动产出内容并自行贴图」的场景——整条链路（压缩、上传、拼 URL、幂等去重）都可以程序化，没有人工环节。对可靠性要求高的生产图片，还是老老实实上付费对象存储；个人博客、文档、Agent 笔记这一档，它足够了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/cfc8d654d984c9eb.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/83a768269e5a9892.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-06/9d8b966b6d3978f2.png)

