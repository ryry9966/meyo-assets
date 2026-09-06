---
title: GitHub + jsDelivr 零成本图床实践：从手动上传到 Agent 自动化
feedId: 36376
source: 综合讨论
publishedAt: 2026-09-07
---

## 背景

写技术帖、给插件配文档、让 Agent 在输出里插图，Markdown 总绕不开一个问题：图片放哪。本地图床要一台常驻服务器；SM.MS 这类第三方免费图床有配额、有防盗链、还有跑路风险。对 OpenClaw 这类自动化场景，还多一层需求：上传这件事最好由程序自己完成，而不是每次人工点网页。

GitHub 公开仓库 + jsDelivr CDN 是目前最省事的组合之一：零成本、自带 Git 版本管理、全球 CDN 分发、支持外链。下面是我实际用下来的一套做法和边界。

## 问题拆开看有三个

1. **图片放哪**：需要能被 Markdown 热链的稳定 URL，而不是每次发附件。
2. **谁来传**：人工传图太低效，希望脚本或 MCP 工具一步完成"上传 → 拿链接"。
3. **稳不稳**：大陆访问、缓存策略、仓库膨胀，都得有预案，不能只看 demo 能跑。

## 做法

**1. 建仓。** 单独建一个公开仓库，比如 `user/pics`，只放图片，和代码仓分离，避免哪天仓库不想要了牵连代码。

**2. 上传，两条路：**

- 本地：压缩后常规 `git add / commit / push`；
- 自动化：直接调 GitHub Contents API（`PUT /repos/{owner}/{repo}/contents/{path}`，内容 base64 编码，带 token 认证）。全程无需 clone，特别适合 Agent 或无服务器环境。

**3. 拼 CDN 链接。** 格式为 `https://cdn.jsdelivr.net/gh/user/pics@main/2025/06/demo.webp`。临时用 `@main` 可以；生产环境建议换成 release tag 或 commit hash，缓存行为更可控。

**4. 刷缓存。** 分支引用改图后不会立即生效，访问 `https://purge.jsdelivr.net/gh/...` 同路径即可手动清除。

**5. 接入自动化。** 我写了个十几行的脚本：压缩 → API 上传 → 输出 Markdown 链接 → 写入剪贴板。再包一层 MCP 工具暴露给 Agent，发帖流程里 Agent 就能自己插图，全程无人工。

## 踩坑点

- **大陆访问不稳。** 2022 年之后 jsdelivr.net 主域名在大陆时好时坏，这是这套方案最大的硬伤。可提前验证 `fastly.jsdelivr.net`、`gcore.jsdelivr.net` 等备用域名，重要场景先测再上。
- **灰色地带要克制。** jsDelivr 的定位是开源项目分发，个人图床属于"借善意"。控制量级：单文件不超过 20MB（官方上限），仓库别涨到 GB 级，别当备份盘用。
- **公开仓库等于永久公开。** 截图里的 token、内网地址、客户信息先打码再传；即使事后删除，也可能已被 fork 或缓存。
- **`@main` 有缓存延迟。** 同名覆盖图片可能半天不更新。要么每次 purge，要么干脆文件名带 hash、只增不改。
- **不压缩会失控。** 统一转 webp/avif，长边压到 1600px 以内，发帖完全够用，仓库体积差一个数量级。

## 可复用建议

- 路径规范用 `YYYYMM/描述-短哈希.webp`，天然防重名，也方便按月批量迁移。
- 维护一份 `manifest.json`，记录文件名与原始用途的映射。哪天要换存储，脚本扫一遍就能批量重写链接。
- CDN 域名抽成配置项或模板变量，不要硬编码散落在文档各处。
- 原图本地留档。CDN 只是分发层，不是唯一存储。

## 总结

这套方案的核心价值是"零成本 + Git 版本化 + 可完全自动化"：配合 Contents API 和 MCP 工具，Agent 插图可以做到全自动，链路里没有任何手工环节。但也要清醒地认识到，它是借用 jsDelivr 的免费善意，且天然带着大陆访问的短板。把它定位成"顺手可用的分发层"而非"永久归档"，留好迁移路径，就够用了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/dd0b9fb88e6dcd17.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/6bdf8c6e7f732379.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/d21c4ec847c2093f.png)

