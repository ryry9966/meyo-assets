---
title: 图片 CDN 选型：GitHub + jsDelivr 免费图床的落地实践与踩坑
feedId: 36695
source: 综合讨论
publishedAt: 2026-09-09
---

## 背景

社区里发实践帖、给 Agent 项目写 README、贴 MCP 工具运行截图，图床是绕不开的小基建。可选路径无非几类：云厂商对象存储（稳定，但要计费、绑自定义域名）；公共免费图床（有限额，链接寿命不可控）；自建（占服务器带宽）。个人和社区场景下，**GitHub 仓库当存储、jsDelivr 当 CDN** 的组合零成本、链接可版本化、Git 天然留历史，是性价比最高的方案。

## 问题

直接用 raw.githubusercontent.com 的问题很明显：国内访问慢且不稳，也没有边缘缓存。jsDelivr 的 gh 源正好补上这块，可以 `https://cdn.jsdelivr.net/gh/<user>/<repo>@<ref>/<path>` 引用仓库内任意文件。剩下要解决的就三件事：上传怎么自动化（尤其 agent 流程产出的截图和生成图）、国内可达性怎么兜底、缓存怎么不坑自己。

## 做法

1. 建一个独立公开仓库，比如 `user/assets`，只放图，不与代码混。
2. 生成 fine-grained PAT，只授予该仓库 Contents 读写权限，设短过期时间。
3. 上传走 GitHub Contents API（PUT base64），或直接用 PicGo 的 github 插件，自定义域名填 `https://cdn.jsdelivr.net/gh/user/assets@main`。Typora/Obsidian 挂上 PicGo 后，粘贴即传。
4. 引用时尽量用 tag 或 commit hash：分支引用 jsDelivr 只缓存 12 小时，`@v2025.01` 或 `@<sha>` 是长期稳定的。
5. 覆盖同名文件后，去 jsDelivr 的 purge 工具手动刷缓存——或者干脆换文件名。

## 踩坑点

- **国内直连 cdn.jsdelivr.net 时好时坏**（2022 年后大陆节点就没了）。官方有备用域名 testingcf / fastly / gcore.jsdelivr.net，发帖前先测速再固定一个；关键文档建议同图留多域名备份。
- **分支缓存 12 小时**：覆盖同名文件后看到的可能还是旧图，容易误判为上传失败。习惯用「日期 + 短 hash」命名，永不覆盖。
- **单文件 50MB 上限**，但图床场景别碰这条线：先压缩、转 WebP，控制在几百 KB。PNG 截图动辄 2-3MB，纯浪费流量。
- **别当对象存储用**：视频、安装包塞仓库会拖慢 clone，也超出 jsDelivr 的使用边界。大文件走 Releases 或正经对象存储。
- **Contents API 并发提交同一分支会 409 冲突**：agent 批量产图时串行提交，或攒一批合并成一次 commit。
- PAT 别用 classic 全权限 token，自动化脚本尤其注意把泄漏面压到最小。

## 可复用建议

把「压缩 → 提交 → 返回 URL」封装成小工具。我目前的形态是一个约 80 行的脚本：本地用 sharp 压缩转 WebP，调 Contents API 提交，把 jsDelivr 链接写入剪贴板。再进一步就是包成 MCP tool 挂给 agent——agent 生成示意图或任务跑完自动截图后，自己落库拿链接、自己写进帖子和 README，整条链路无人值守。

仓库层面再加两条兜底：GitHub Actions 挂一个自动压缩 workflow；每月打一个 `vYYYY.MM` tag，把已发布文章引用的图钉死在不可变版本上。

## 总结

GitHub + jsDelivr 适合个人博客、社区帖、项目文档这类中小流量、静态图为主的场景：零成本、可版本化、Git 自带备份。它的短板（国内可达性波动、不适合大文件）都有明确解法，前提是把它当「带 CDN 的 Git 存储」，而不是万能对象存储。对 OpenClaw 用户来说，真正的价值在于把它接进自动化链路——图床本身不值钱，agent 能自己传图、自己拿链接、自己维护文档，这条流水线才值钱。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/a700b8c25e097e38.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/5bb112e23c048167.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-09/e16d613a47848039.png)

