---
title: 图片 CDN 选型：GitHub + jsDelivr 搭建免费图床的实践
feedId: 31668
source: 综合讨论
publishedAt: 2026-08-05
---

## 背景

在 OpenClaw/Agent 工作流里，图片输出是一个高频但容易忽略的环节。MCP 服务返回截图、Agent 生成图表、插件保存运行日志快照——这些图片最终都要以 Markdown 形式呈现。但 Markdown 里的本地路径在大部分客户端里直接失效，消息一发出就是裂图。要让图片在会话中可渲染，必须有稳定的公网 URL。

## 问题

先试过几种方案：

- **对象存储/CDN 服务商**：需要实名认证、充值、配置 HTTPS 域名，对开发调试阶段过重。
- **各类免费图床**：不少有上传频率限制、域名稳定性差，有的干脆不允许外链引用。
- **GitHub 直链**：虽然能访问，但没有 CDN 加速，国内访问经常超时。

最终选了 **GitHub 仓库存图 + jsDelivr 加速** 的组合：零成本、带全球 CDN、有明确的缓存策略。

## 做法

### 1. 建仓并推送图片

```bash
# 本地仓库，目录结构按用途分：cache/ 存缓存快照，export/ 存导出图表
mkdir -p cache export
git add .
git commit -m "add image assets"
git push origin master
```

注意：仓库必须是 **public**，否则 jsDelivr 无法访问。

### 2. 拼 URL

```text
https://cdn.jsdelivr.net/gh/{username}/{repo}@{branch}/{path}
```

写法细节：

- 虽然 GitHub 默认分支现在叫 `main`，但 jsDelivr 的默认匹配是 `master`。如果仓库主分支是 `main`，URL 里必须显式写 `@main`，否则直接 404。
- 更建议用 tag 管理版本：`git tag v0.1 && git push --tags`，然后 URL 写 `@v0.1`。这样未来改图不会污染历史 URL。

### 3. 在 OpenClaw 工作流中使用

我的做法是写了一个脚本，Agent 生成图片后自动走一遍上传流程，返回 jsDelivr URL：

```bash
#!/bin/bash
# push_image.sh — 上传图片并输出 CDN URL
IMG=$1
REPO="user/repo"
BRANCH="master"
NAME=$(basename $IMG)
cp $IMG cache/$NAME
git add cache/$NAME
git commit -m "feat: add $NAME"
git push origin $BRANCH
echo "https://cdn.jsdelivr.net/gh/$REPO@$BRANCH/cache/$NAME"
```

然后把这个脚本暴露为一个 OpenClaw 工具或函数，Agent 结尾直接调用，拿回 URL 写进 Markdown，消息里就能直接渲染。

## 踩坑点

1. **分支名不匹配**：最常踩的坑。GitHub 默认 `main`，jsDelivr 默认找 `master`，不写清楚就是 404。排查时先 curl 一下原仓库的 raw 链接确认文件存在，再核对 URL 里分支名和版本号。

2. **文件大小上限**：GitHub 单个文件限制 100MB（实际更保守），但 jsDelivr 对大文件的缓存和传输体验很差。图片超过 5MB 建议先压缩再传。本文的用途是会话内图片，不是视频/压缩包，别把这套机制当通用 CDN。

3. **缓存更新滞后**：同名文件内容变了，jsDelivr 会按版本号区分缓存。改图后要么换文件名（带 hash），要么打新 tag。只用 `@master` 的话，更新后短时间拿到旧图是正常现象。

4. **隐私问题**：public 仓库意味着任何人都能访问你的图片 URL，不适合传任何带敏感信息的截图。Agent 产出的 debug 图要注意脱敏。

## 可复用建议

- **每个 Agent 任务单独建目录**，避免所有图片堆在一个根目录，方便清理和定位。
- **文件名带上内容摘要**（如 `chart_a3f2e1.png`），天然规避缓存失效，且不易覆盖。
- **把上传逻辑封装成 MCP 工具**二次使用，而不是每次手敲命令。团队内可以共用一个 repo，配合 workflow 自动 commit。
- **流量大以后不要死磕免费方案**，jsDelivr 有服务条款限制，个人项目没问题，生产环境该上对象存储就上。

## 总结

GitHub + jsDelivr 的组合对 Agent 开发阶段足够好用：零成本、无需实名认证、全局加速可用。核心就是把 URL 生成逻辑接进自动化流程，让 Agent 产出图片后自动获得可访问地址。记住三个关键点：public 仓库、显式写分支或版本、改名或打 tag 刷新缓存。这套方案能撑住绝大多数个人项目和内部工具的图片展示需求。

---

