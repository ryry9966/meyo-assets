---
title: 图片 CDN 选型：GitHub + jsDelivr 搭建免费图床的实践
feedId: 36384
source: 综合讨论
publishedAt: 2026-09-07
---

## 背景

写技术帖、文档，或者让 Agent 产出带图报告时，贴图永远是个小麻烦。常见方案各有硬伤：对象存储要花钱、个人 VPS 带宽窄还可能要备案、公共图床说挂就挂、微信图片有防盗链。OpenClaw 社区里经常要分享工作流截图、流程图、MCP 配置演示，一个免费、免运维、可自动化的图床是实际需求。我目前的方案是 GitHub 仓库 + jsDelivr，用了一年多，把经验和坑整理如下。

## 问题

核心诉求四条：图片能拿到直链、国内访问尽量可用、零成本零维护、上传环节能被脚本或 Agent 接管。市面上没有哪个服务四条全占，只能自己组合。

## 做法

1. **建仓**：新建一个公开仓库如 `openclaw-assets`，只放图片，按 `2025/01/语义名.webp` 分目录。
2. **引用**：走 jsDelivr 的 gh 源：

```
https://cdn.jsdelivr.net/gh/<user>/<repo>@<version>/<path>
```

`@main` 始终指向最新；要不可变链接就钉 commit hash，jsDelivr 对固定版本的缓存是长期的。
3. **刷新缓存**：改图后访问 `https://purge.jsdelivr.net/gh/...` 手动 purge，分支链接缓存约 12 小时。
4. **自动化**：写个 30 行左右的脚本走 GitHub Contents API：本地文件 → 压缩转 WebP → 上传 → 返回 CDN 链接并复制到剪贴板。再把它包成 MCP tool 或 agent skill，之后 Agent 写帖时可以直接输出图片链接。不想写脚本可以用 PicGo 这类现成客户端。
5. 可选：加一个 GitHub Action，push 时自动用 sharp 压图再提交，控制仓库体积。

## 踩坑点

- **国内可用性有波动**。jsDelivr 在大陆时不时抽风，别把它当关键基础设施。重要文档我留双链备用：`fastly.jsdelivr.net`、`gcore.jsdelivr.net` 前缀可直接替换。
- **缓存不即时**。`@main` 更新后最长 12 小时不生效，要么 purge，要么换 commit hash。高频更新的图不要用分支链接。
- **别用 Git LFS**。jsDelivr 不回源 LFS 文件，你会拿到一个指针文本文件。
- **仓库必须公开**，等于图片人人可见。上传前检查截图里有没有 token、内网地址、登录态。我的习惯是敏感图一律本地打码后再入仓。
- **文件名全小写加连字符**，中文和空格会带来 URL 编码问题，Agent 按固定规则生成路径最省心。
- PNG 转 WebP 通常省 60% 以上体积，原图不要进仓库。

## 可复用建议

- 把「压缩 → 上传 → 回链」做成一个 skill，是这套方案对 OpenClaw 用户最大的价值：说一句"把这张图发出去"，全流程自动完成。
- 仓库即真相，CDN 只是缓存层，哪天换 CDN 只是改一个 URL 前缀。
- jsDelivr 的定位是开源项目文件分发，个人博客和社区帖的量级完全没问题，但别拿去当大文件或视频床，公平使用条款在那儿。

## 总结

这套方案本质是「GitHub 存储 + jsDelivr 加速 + 自建脚本自动化」。它不承诺生产级的确定性，但对社区分享、博客配图、Agent 自动产出这些场景，是成本和维护量最低的组合。先用起来，等稳定性和量级要求上来再迁对象存储，届时迁移成本也只是一个前缀替换。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/7a4f83ccc142cd6c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/5555f5a5eed42f8b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-07/1761b97d4a979788.png)

