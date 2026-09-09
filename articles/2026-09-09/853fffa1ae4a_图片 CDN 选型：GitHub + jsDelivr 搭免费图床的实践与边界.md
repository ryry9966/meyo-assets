---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的实践与边界
feedId: 36767
source: 综合讨论
publishedAt: 2026-09-09
---

## 背景

做 Agent / 插件开发时经常遇到图片外链需求：机器人产出的图表要贴进网页、文档截图要能被 Markdown 直接引用、自动化流程生成的图片要给下游服务消费。这类场景的共同要求是：可热链、够稳定、零成本、最好走 CDN。

## 问题

- `raw.githubusercontent.com` 没有 CDN，国内访问不稳定，不适合当生产外链；
- 各类免费图床普遍有防盗链，且跑路风险高；
- 对象存储要备案、绑域名、付费，对个人项目偏重。

折中方案：GitHub 仓库做存储层，jsDelivr 做分发层。但要清楚它的适用边界，后面会讲。

## 做法

1. 建一个公开仓库（如 `assets`），按 `img/2024/06/` 或项目名分目录；
2. 引用格式：`https://cdn.jsdelivr.net/gh/<user>/<repo>@<ref>/<path>`，`ref` 用分支或 tag 均可；
3. 上传自动化：小脚本调 GitHub Contents API（PUT `/repos/{owner}/{repo}/contents/{path}`），携带 base64 内容与文件 sha，配 fine-grained PAT。懒得写就用 PicGo；
4. 压缩先行：sharp 压一遍，PNG 转 WebP，单文件控制在几百 KB；
5. 在 OpenClaw 侧包成 MCP 工具：输入本地图片路径，返回 CDN URL，Agent 就能把外链直接写进回复或文档。

## 踩坑点

1. **缓存污染**：jsDelivr 对同名路径缓存很长，覆盖上传后老 URL 可能还是旧图。解法是文件名带内容 hash（`20240612-a3f9c2.png`），永远写新路径；紧急情况去 `purge.jsdelivr.net` 手动刷。
2. **大文件不服务**：jsDelivr 只分发约 20MB 以内的文件，且仓库不宜当备份盘撑大，图片场景够用，但要克制。
3. **国内连通性波动**：jsDelivr 在大陆可用性时有起伏，不能当唯一通道。关键页面建议做 fallback 链：jsDelivr → raw → Statically，或前端 `onerror` 切换。
4. **政策风险**：jsDelivr 定位是开源项目分发，不是个人存储。纯当私人网盘用，有仓库被 suspend 的先例。建议控制在博客/文档配图这类"分发"用途，量级适度。
5. **Contents API 并发冲突**：多进程写同一路径会 409/422。串行化上传，或失败后重新 GET 拿最新 sha 重试。
6. **公开即公开**：仓库内容任何人可见，含敏感信息的截图（token、内网地址）先打码再传。

## 可复用建议

- 命名规范统一为 `{yyyymmdd}-{8位hash}.{ext}`，天然规避缓存问题；
- 维护一份 `manifest.json`（原名 → CDN 路径映射），将来迁对象存储只需改 base URL；
- 上传逻辑封装成 MCP tool 或 CLI 子命令，参数就三个：文件路径、目标目录、返回格式；
- PAT 用 fine-grained token，只授权这一个仓库，Contents 读写即可，最小权限。

## 总结

GitHub + jsDelivr 是"零成本 + 可自动化"约束下的合理解，尤其适合 Agent 产出物外链、个人博客和项目文档。它的短板在连通性波动与政策边界，所以设计上要预留切换成本：内容寻址的命名、manifest、fallback 链。图床本身不是核心资产，可迁移性才是。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/11a4e88c5f784df5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/f0203198df55f927.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/a0838f3d4162edef.png)

