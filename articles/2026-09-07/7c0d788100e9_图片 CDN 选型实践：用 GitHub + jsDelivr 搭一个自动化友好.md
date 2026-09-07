---
title: 图片 CDN 选型实践：用 GitHub + jsDelivr 搭一个自动化友好的免费图床
feedId: 36411
source: 综合讨论
publishedAt: 2026-09-07
---

## 背景

在 OpenClaw 的日常使用里，图片是绕不开的资源类型：Agent 生成的报告要贴架构图，MCP 工具的输出要截图佐证，社区帖子和博客也需要长期有效的外链。公共图床这几年来来去去去，限速的限速、挂链的挂链，指望第三方托管不如自己控制源头。我的方案最终落在了「GitHub 仓库 + jsDelivr CDN」这个组合上，用了小半年，这里把实践和坑完整记录一下。

## 问题

直接用 `raw.githubusercontent.com` 有两个硬伤：国内访问不稳定，且对匿名请求有速率限制，不适合做长期外链。jsDelivr 对 GitHub 公开仓库做了全球 CDN 缓存，正好补上这块。我的核心诉求有三条：

- 免费，且不依赖某家随时会跑路的图床服务
- URL 格式可预测，方便脚本和 Agent 自动拼接
- 能接入 OpenClaw 工作流，上传图片后直接拿到可用外链

## 做法

1. **建仓库**：新建一个公开仓库如 `cdn-assets`，目录按 `img/2024/` 这类规则组织，保持扁平。
2. **上传方式三选一**：本地 git push、GitHub Contents API（base64 上传，适合脚本）、或 GitHub Actions 监听目录自动发布。
3. **拼外链**：格式为 `https://cdn.jsdelivr.net/gh/<user>/<repo>@main/img/xxx.png`。把 `@main` 换成 commit hash 或 tag，URL 就成为不可变资源，缓存永不失效。
4. **自动化**：给 OpenClaw 写一个 MCP 工具，输入本地图片路径，内部用 Contents API 上传，返回拼好的 jsDelivr URL，几十行代码。之后 Agent 写文档时可以自助贴图，整条链路不需要人介入。
5. **缓存刷新**：确需覆盖同名文件时，请求 `https://purge.jsdelivr.net/gh/...` 强制回源。

## 踩坑点

- **同名覆盖不生效**：jsDelivr 对 GitHub 文件缓存周期较长，覆盖后旧图可能持续返回数天。结论是永远别覆盖，文件名用内容 hash（如 sha1 前 8 位），天然幂等，还能省掉 purge。
- **20MB 文件上限**：jsDelivr 不服务超过 20MB 的 GitHub 文件，上传前先压缩，能转 WebP 就转。
- **仓库定位要清楚**：GitHub 建议单仓库 1GB 以内，图床不是对象存储，堆视频和大型源文件迟早出事。
- **国内连通性波动**：jsDelivr 在国内的可用性时好时坏，我在脚本里预留了 `fastly.jsdelivr.net` 等备用域名。选型时别把它当唯一依赖。
- **Token 权限收敛**：用 fine-grained PAT，只授这个仓库的 Contents 读写，别把全权限 token 塞进自动化脚本。匿名调用 GitHub API 每小时仅 60 次，务必带 token（5000 次/小时）。

## 可复用建议

- **hash 命名 + commit pin**：URL 不可变后，浏览器和 CDN 缓存全是正向收益。
- **封装成 MCP 工具或 OpenClaw skill**：这是整套方案收益最大的一步；顺手在仓库维护一份 `index.json` 资产清单，方便 Agent 检索已有图片。
- **上传前压缩**：500KB 以内的 WebP 足够文档和帖子使用。
- **敏感内容不进公开仓库**：图床默认全网可读，当它是公共发布渠道而不是网盘。

## 总结

GitHub + jsDelivr 不是性能最强的图床方案，但免费、稳定、URL 可预测。配合 OpenClaw 自动上传后，「生成图片 → 拿到外链 → 写进文档」可以完全无人参与。对个人博客、技术文档和社区帖子的流量规模，它完全够用。把它当边缘分发节点而不是数据库，不追求 SLA，就不太会踩坑。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/b369ba7a81a4d3be.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/40108d40885030f1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/6856b6265d4290b4.png)

