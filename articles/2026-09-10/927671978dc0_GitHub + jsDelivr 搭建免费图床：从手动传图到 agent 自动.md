---
title: GitHub + jsDelivr 搭建免费图床：从手动传图到 agent 自动上传的实践
feedId: 36913
source: 综合讨论
publishedAt: 2026-09-10
---

## 背景

写技术文档、博客、插件 README 时，图片托管是个绕不开的小事。可选方案无非几种：对象存储要付费或绑域名，第三方免费图床跑路风险高，GitHub raw 链接在国内加载慢且不稳定。GitHub 仓库 + jsDelivr CDN 是折中里比较耐用的一个：仓库即备份，CDN 即加速，成本为零。对 OpenClaw 用户来说，这套流程还有个额外价值——它足够简单，可以完整交给 agent 自动化。

## 问题

直接贴 `raw.githubusercontent.com` 链接有三个毛病：

- 国内访问经常超时；
- 无 CDN 缓存，多人打开同一个 README 时加载慢；
- 部分场景下 MIME 类型不正确，浏览器触发下载而不是渲染。

目标是：拿到一个稳定、可缓存、可直接嵌入 Markdown 的图片 URL，并且能被脚本或 MCP 工具自动生成。

## 做法

1. 建一个**公开**仓库（如 `images`），必须 public，私有仓库 jsDelivr 拒绝服务。
2. 按 `yyyy/mm/` 目录归档，文件名只用 ASCII 字符。
3. URL 格式：

```
https://cdn.jsdelivr.net/gh/<user>/images@main/2025/01/demo.png
```

`@main` 指分支，也可换成 release tag（如 `@v1.2`）获得不可变 URL。

4. 自动化（对 agent 用户最有价值的部分）：写一个几十行的脚本或 MCP tool，流程是——本地图片 → 调 GitHub Contents API（`PUT /repos/{owner}/{repo}/contents/{path}`，body 放 base64）→ 拿到 commit 结果 → 拼出 jsDelivr URL 返回。注册成 OpenClaw 的 skill/MCP 工具后，agent 写文档时可以自己跑完"截图 → 上传 → 插入链接"整条链路，不用人手点网页。

## 踩坑点

- **缓存不刷新**：同名覆盖后 CDN 仍返回旧图。两个办法：文件名带内容 hash（`demo_a1b2c3.png`），或主动调 purge 接口 `https://purge.jsdelivr.net/gh/...`。推荐前者，简单可靠。
- **单文件上限**：jsDelivr 对 gh 来源的文件约 20MB，超限直接失效。截图和示意图够用，别放原图和视频。
- **仓库体积**：建议控制在 1GB 内。GitHub 对超大仓库有警告，jsDelivr 也可能对"纯网盘式"滥用限流。控制上传频率，别当存储盘用。
- **国内可用性波动**：jsDelivr 主域名在大陆的可达性历史上反复过，目前一般可用，但别单点依赖。备用镜像 `fastly.jsdelivr.net`、`gcore.jsdelivr.net`，换 URL 前缀即可，仓库不用动。
- **默认分支名**：老仓库默认分支可能是 `master`，URL 写 `@main` 会 404，先确认。

## 可复用建议

- 命名规范：`日期-描述-短hash.png`，天然规避缓存问题，也方便 agent 按时间回溯。
- 上传脚本加两个细节：失败重试一次；返回结果同时给 raw 链接和 jsDelivr 链接，raw 用于排障。
- 把 purge 调用包进脚本，提供 `--purge` 参数。
- 多人协作时，每人或每项目一个仓库，避免权限互相牵扯。

## 总结

GitHub + jsDelivr 适合个人博客、文档站、插件 README 这类中低流量场景：零成本、有版本备份、URL 稳定，配合脚本或 MCP 工具可以完全自动化。局限也明确——有大小限制、国内可用性需要备选镜像、不适合承载 SLA 级业务。把它当成"文档基础设施"而不是"生产 CDN"，期望就对了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/649267dc5d756e4d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/a0b16ac82280bf9f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-10/1ccc21c2ab3bde09.png)

