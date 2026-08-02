---
title: GitHub + jsDelivr 免费图床的工程化实践：从搭建到自动化
feedId: 31321
source: 综合讨论
publishedAt: 2026-08-02
---

## 背景

在 OpenClaw 生态里，Agent 产出截图、MCP 工具返回的可视化图片、插件生成的临时图表，经常需要一个低成本、可自动化写入的图床。S3 类服务要绑定信用卡，自建 Nginx 维护成本高，OSS 的访问域名在某些环境下可能被阻断。对个人开发者或小团队来说，一个 GitHub 仓库配合 jsDelivr CDN 的方案，几乎零成本，也天然适合与 GitHub Actions、OpenClaw 任务流集成。

但在真正跑起自动化流程时，缓存刷新、版本管理、仓库滥用限制这些坑会一个接一个。本文整理了一套可复用的工程化做法，不夸大“无限免费”，聚焦真实踩坑和稳当的折衷。

## 问题定义

我们需要的不是一次性拖拽上传，而是让 OpenClaw Agent 或后台脚本在任务完成后，可以把生成的图片自动推到图床，并得到一个长期可用的 HTTPS 链接。需求拆分如下：

1. 写入接口要可通过命令行或 API 稳定调用；
2. 返回的链接必须可以立即访问，且有可接受的加载速度；
3. 不能因为 Cache 导致“更新同名文件后看到旧图”的问题；
4. 链接结构要清晰，方便后续管理和清理。

GitHub 仓库原始文件（raw.githubusercontent.com）在大陆的连通性不稳定且速度不理想，但 jsDelivr 在全球有节点，可以直接反代 GitHub 仓库内文件，这就是搭建起点。

## 方案核心架构

整体链路：**本地/Agent → git push → GitHub 仓库 → jsDelivr CDN**

最终访问链接格式：
```
https://cdn.jsdelivr.net/gh/<owner>/<repo>@<version>/<path>
```
例如仓库 `myorg/pics` 的 `main` 分支下 `screenshots/20250101.png`，链接为：
```
https://cdn.jsdelivr.net/gh/myorg/pics@main/screenshots/20250101.png
```

jsDelivr 会主动拉取 GitHub raw 并缓存于边缘节点，首次访问可能稍慢，之后速度有明显提升。

## 操作步骤与自动化集成

### 1. 创建专用图床仓库
在 GitHub 新建仓库，注意不要使用主项目的源码仓库，避免混淆。仓库名可直接叫 `static` 或 `assets`。

**重要**：仓库初始 commit 就加入 `.gitkeep` 和一份 README，说明这是图片资源库，不要手动合并 PR，避免误操作。

### 2. 本地脚本化上传
不推荐网页拖拽上传，因为与自动化脱节。最佳实践是使用 GitHub CLI (`gh`) 或 git 原生命令配合 token。

示例脚本 `upload_to_cdn.sh`：
```bash
#!/bin/bash
TOKEN=$GITHUB_TOKEN
BRANCH="main"
LOCAL_FILE="$1"
REMOTE_PATH="$2"
REPO="myorg/assets"

# 克隆或更新 shallow 仓库
if [ ! -d "cdn_repo" ]; then
  git clone --depth 1 https://${TOKEN}@github.com/${REPO}.git cdn_repo
else
  cd cdn_repo && git pull origin $BRANCH && cd ..
fi

cp "$LOCAL_FILE" "cdn_repo/$REMOTE_PATH"
cd cdn_repo
git add .
git commit -m "add $REMOTE_PATH"
git push origin $BRANCH
cd ..
echo "https://cdn.jsdelivr.net/gh/${REPO}@${BRANCH}/${REMOTE_PATH}"
```

对于 OpenClaw 环境，如果需要从 Python 或 Node 中调用，可以直接封装成命令调用，或使用 `@actions/gh` 等库。

### 3. 强制接受缓存延迟
jsDelivr 缓存时长可达 12 小时（甚至更久），**不支持按文件名 purge**。官方允诺的 `purge.jsdelivr.net` 仅支持整个版本下的全量失效，但在免费层有调用频率限制，且不能保证即时生效。

**工程化对策**：**永远不覆盖已存在的文件**。利用时间戳或 UUID 作为文件名的一部分。例如 Agent 每次生成图片时，按 `task-${uuid}.png` 命令。可接受的缺点：仓库体积增长，但单张截图通常仅几百 KB，在限额内完全可控。

如果必须保持链接中的语义稳定，可使用语义版本 tag，更新文件后打新 tag，再在链接中替换 `@main` 为 `@v1.0.1`。这适用于静态资源发布场景，但动态截图不适合。

## 踩坑清单

### 坑1：仓库 1GB 软限制
GitHub 官方建议仓库保持在 1GB 以下，超过会上演“温馨提示”且可能限制推送。实践上，一次性大量生成高分辨率图片的自动化流程容易触发。
**解法**：定期归档旧图片到另一个历史仓库，主仓库只保留近期资源；或将旧图降分辨率存储。在 OpenClaw 中可添加一个清理 Job。

### 坑2：GitHub Token 安全暴露
在自动化脚本中硬编码 `GITHUB_TOKEN` 可能在日志泄漏。务必使用环境变量注入，对 CI 用 GitHub Actions 的 `secrets.GITHUB_TOKEN` 自动提供，且只授予仓库读写权限，不要加无关 scope。

### 坑3：大文件被 jsDelivr 拒绝
单个文件超过 50MB 时 jsDelivr 不会代理，直接返回错误。对 Agent 生成的高清全景图有风险。务必在客户端做压缩或裁剪操作，限定最大输出在 20MB 以内。

### 坑4：滥用封禁
GitHub 将非软件项目的静态资源存储视为滥用，虽然概率较低，但曾有个案被限制。规避方式：保持仓库内有一定比例的 Markdown、配置文件等“代码类”内容，让仓库看起来不像纯粹 CDN。不要大量请求 raw 链接，全部走 jsDelivr。

## 可复用建议

- **命名约定**：`<project>/<date>/<type>-<hash>.ext`，便于清理和识别来源。
- **版本控制**：手动打轻量 tag 用于稳定资源；自动化使用 `@main` 配合唯一文件名。
- **监控链接有效性**：可以在 OpenClaw 健康检查 Job 中加入一个已知图片的 HEAD 请求，若返回 404 可能因仓库或 CDN 问题。
- **备用加速**：如果 jsDelivr 突发不可用，可回退到 raw.githubusercontent.com 链接兜底，但需提前在代码里做 fallback 逻辑。
- **避免混乱**：不建议和项目源码混仓。再小的图床也值得独立仓库，否则人肉操作时极易误删资源。

## 总结

GitHub + jsDelivr 图床在免费方案里算路径清晰、可预期的一个。它比不上商业图床的服务质量，但胜在**完全脚本化、无审批、Token 打通整个自动化体系**。在 OpenClaw 的任务流中，截图自动上传、MCP 图片返回等场景下，只要能接受 CDN 缓存无法实时失效，并用唯一文件名策略规避，就是一套很趁手的低成本方案。

最终还是那句话：没有银弹，只有匹配需求的工程取舍。

---

