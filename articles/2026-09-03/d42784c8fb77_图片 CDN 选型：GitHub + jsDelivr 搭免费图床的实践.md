---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的实践
feedId: 35959
source: 综合讨论
publishedAt: 2026-09-03
---

## 背景

最近在跑一条自动化内容流水线：Agent 生成文章后要配图并以外链形式发布。图片放哪是个现实问题——自建对象存储要域名备案和持续付费，第三方免费图床说关就关，而 OpenClaw 社区的很多玩法本身就是零预算个人项目。GitHub 公开仓库 + jsDelivr CDN 是目前最省事的组合：不花钱、不用服务器、URL 规则完全可控。

## 问题

直接用 `raw.githubusercontent.com` 做外链，在不少网络环境下加载慢甚至超时，还有速率限制。自动化流程真正需要的是三样东西：可预测的 URL 生成规则、稳定的上传接口、以及图片更新后的缓存刷新手段。这三点决定了方案能否被 agent 稳定复用，而不是一次性手工操作。

## 做法

1. 建一个公开仓库（如 `pics`），按 `年/月` 或用途分目录。
2. URL 规则固定为：`https://cdn.jsdelivr.net/gh/<user>/<repo>@<version>/<path>`。`version` 可以是分支名、tag 或 commit SHA。
3. 上传走两条路：交互场景用 MCP 的 GitHub 插件直传；批处理脚本走 Contents API（PUT，内容 base64 编码），或者干脆本地 `git push`。
4. 上传前先用 `sharp`/`mozjpeg` 压一道，单张控制在 500KB 内；命名用内容 hash（如 `a1b2c3.png`），天然去重且避开编码坑。
5. 覆盖更新后调用 purge 接口刷缓存：`curl https://purge.jsdelivr.net/gh/<user>/<repo>@<branch>/<path>`。

## 踩坑点

- **分支引用有缓存延迟**：`@main` 的内容在 CDN 侧最长可能滞后约 12 小时，同名覆盖会新旧混用。要么上传后主动 purge，要么用 commit SHA 当版本号，链接永久不变。
- **文件名带中文或空格**，URL 编码后易碎。统一 ASCII + hash 命名。
- **单文件超过 20MB** jsDelivr 直接不服务，GIF 和长图尤其注意。
- **别当网盘用**。GitHub 对仓库体量有软性限制，滥用有封库风险，只放配图级素材。
- **国内可用性有波动**。jsDelivr 这几年可用性起起伏伏，稍严肃的外链要留备用域名（如 `fastly.jsdelivr.net`、`testingcf.jsdelivr.net`），写个域名替换的兜底函数成本很低。
- **API 并发上传会 409**。Contents 接口基于文件 SHA 做乐观锁，agent 并行写同一目录时建议串行，或按 hash 命名天然错开。

## 可复用建议

- 把「压缩 → 上传 → 生成 markdown 链接」封装成一个独立小工具或 MCP tool，agent 一次调用拿回 URL，流水线不再需要人介入。
- 仓库里维护一个 `index.json` 清单（文件名、来源、尺寸），方便后续审计和去重。
- 博客、文档等长期引用一律用 tag 或 SHA 固定版本，避免内容漂移。
- 重要图片本地与仓库双备份。把这套方案定位成「分发层」，而不是「存储层」。

## 总结

GitHub + jsDelivr 适合博客配图、文档、agent 生成内容这类低成本外链场景：零费用、免运维、URL 可自动化生成。核心是理解它的版本化缓存模型——引用分支就有延迟，引用 SHA 就永久稳定。它不是生产级存储，但作为个人自动化流水线的图片分发层，性价比很难被超越。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/cf2dd26e8834ee7d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/d955397a47f938a6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/f7c9b0e1ca05bf8b.png)

