---
title: 用 GitHub + jsDelivr 搭一套工程化免费图床
feedId: 31829
source: 综合讨论
publishedAt: 2026-08-06
---

## 背景

在 OpenClaw、MCP 插件或日常自动化脚本的输出里，经常需要生成可公开访问的图片链接——截图存证、OCR 结果标注、报告图表等。临时传图床要么接口不稳定，要么有调用频率与容量限制；自建 OSS 成本偏高，也不适合个人项目或轻量流水线。

GitHub 公共仓库配合 jsDelivr CDN 可以组成一套零成本、高可用的“图床”。GitHub 提供版本化存储，jsDelivr 提供全球加速与容量支持，组合在一起刚好满足自动化流程中“把生成的图片推上去，拿回一个直链”的需求。本文将记录一套工程化实践，包含上传、链接生成、缓存避坑与 MCP/Agent 集成思路。

## 问题拆解

要把 GitHub + jsDelivr 当成稳定的图片托管，需要解决几个工程问题：

1. **上传自动化**：在脚本或 GitHub Actions 里完成 `git push`，并通过 token 鉴权。
2. **链接规范化**：上传后必须立刻拿到可用的 jsDelivr 直链，不能每次手动拼接。
3. **仓库治理**：防止 `.git` 历史膨胀导致克隆变慢，并避免单仓库文件数量触碰限制。
4. **缓存刷新**：jsDelivr 会长时间缓存同一版本的文件，更新同名图片需有可靠刷新策略。
5. **安全边界**：只适合公开图片，敏感数据绝不能用 public 仓库。

## 实现步骤

### 1. 创建专用图床仓库

在 GitHub 新建 **Public** 仓库，命名如 `static-images`，勾选“Add a README”。建议使用独立账号或组织，避免与业务仓库混淆。本地 clone 后按日期或功能划分目录结构，例如 `2025/04/screenshot-01.png`。

### 2. 配置上传凭据

创建 [Personal Access Token (classic)](https://github.com/settings/tokens)，勾选 `repo` 权限。将 token 存入 CI/CD 的 secret 或本机环境变量 `GITHUB_TOKEN`。

### 3. 编写上传脚本（示例用 Bash）

```bash
#!/bin/bash
set -euo pipefail

REPO="git@github.com:yourname/static-images.git"
BRANCH="main"
DATE_DIR=$(date +%Y/%m)
IMG_PATH="$DATE_DIR/$(date +%s)-${1##*/}"

# clone 浅层仓库避免拖全量历史
rm -rf /tmp/img-upload
git clone --depth 1 --branch "$BRANCH" "$REPO" /tmp/img-upload

cp "$1" "/tmp/img-upload/$IMG_PATH"
cd /tmp/img-upload
git add "$IMG_PATH"
git -c user.name="bot" -c user.email="bot@local" commit -m "Upload $IMG_PATH"
git push https://TOKEN@github.com/yourname/static-images.git "$BRANCH"

# 输出 jsDelivr 链接
echo "https://cdn.jsdelivr.net/gh/yourname/static-images@${BRANCH}/${IMG_PATH}"
```

该脚本会浅克隆、放入图片、提交推送，最后输出标准化直链。`--depth 1` 避免克隆全量图片历史。

### 4. 在自动化流程里调用

**GitHub Actions 用法**：把上传步骤封装成 action，传入图片路径和 token，输出链接供后续 job 使用。也可以部署成 MCP 工具，供 OpenClaw 这类 Agent 在生成报告后直接调用。

**本地或 VPS 使用**：将脚本存为 `/usr/local/bin/gh-img-upload`，`chmod +x` 后即可在任意地方 `gh-img-upload screenshot.png`，快速得到 CDN 链接。

## 踩坑记录

- **jsDelivr 缓存更新延迟**：这是最常见的问题。如果push了新版本的图片使用相同文件名，jsDelivr 仍可能返回旧文件。解决方法：不要覆盖文件名，每次用时间戳或随机串生成唯一路径，彻底避免缓存问题。如果确需覆盖，可以推送后手动调用 purge 接口 `https://purge.jsdelivr.net/gh/yourname/static-images@main/path`，但每次只能清一个文件，且有限速，不推荐用于高频更新。
- **仓库体积膨胀**：图片二进制直接进 Git 会导致历史快速膨胀，`git clone --depth 1` 只能减轻克隆负担，但 GitHub 侧仓库总大小仍在增长。建议定期归档旧月份目录、切分仓库，或使用 Git LFS（但 LFS 存储与带宽需单独计费，会打破免费方案）。一个折中策略是每季度重建仓库，保留脚本中的链接重定向。
- **文件数量限制**：GitHub 对仓库文件数有软性限制，单目录超过约 1000 个文件时 API 性能明显下降。务必按月份甚至按日期分子目录，避免扁平堆积。
- **私有图片误传**：一旦 push 到 public 仓库并被 jsDelivr 缓存，即便立即删除，CDN 边缘节点可能仍留存一段时间。务必在脚本层加入校验：文件命名、尺寸、内容检查，防止泄露截图中的敏感信息。
- **GitHub Token 安全**：建议使用 fine-grained token 且限定单个仓库权限，避免使用主账号全权 token。

## 可复用建议

- **封装为 MCP 工具**：将上传逻辑实现成一个 MCP server，提供 `upload_image` action，自动返回 jsDelivr URL。OpenClaw Agent 在生成图表后可无缝对接，图片直链自动插入最终输出。
- **结合 PicGo 类客户端**：如果不想全自动脚本，也可在 PicGo 或 uPic 中配置 GitHub 图床，每次手动截图后自动上传，日常写文档十分方便。
- **版本化管理**：在 CDN 链接里使用 `@main` 分支名，若要标记稳定版本可用 `@release-v1`，适合需要稳定资源的场景。
- **备用加速**：jsDelivr 在大陆部分地区偶尔被干扰，可准备 fallback 到 statically 等镜像，或同步一份到 Cloudflare R2（免费 10GB 额度）做双链。

## 总结

GitHub 公共仓库 + jsDelivr 构建的免费图床，在非敏感、中低频图片托管场景下足够务实。通过浅克隆、唯一文件名、目录分片和脚本封装，可以做到稳定可靠。对接 OpenClaw 等自动化工具时，一张图片从生成到获得全球可访问的直链，只需一次函数调用。这套方案的优势在于零成本、完全自控、与 Git 工作流天然兼容；代价是需要承担公开仓库的风险，并接受偶尔的 CDN 延迟。选型前务必判断图片是否适合公开访问，如果不满足，还是转向私有 OSS 或带鉴权的方案更合适。

---

