---
title: 面向自动化工作流的免费图床选型：GitHub + jsDelivr 工程化实践
feedId: 31319
source: 综合讨论
publishedAt: 2026-08-02
---

## 背景

在构建 OpenClaw 插件、MCP 服务或自动化脚本时，经常需要生成截图、渲染图表或托管静态资源。现成的图床服务要么收费，要么限制外链次数，要么存在隐私顾虑；自建对象存储又引入额外的运维成本和账单风险。对于非生产环境、低频访问的个人工具链，我们需要一个**零成本、可编程、无外链审核**的图片托管方案。

GitHub 仓库本身就是一个稳定的静态资源托管平台，配合 jsDelivr 全球 CDN，能构建一套完全免费的图床。更重要的是，它可以无缝融入 Git 工作流和 CI/CD，适合自动化场景。

## 为什么选 GitHub + jsDelivr？

- **零成本**：公开仓库无限免费（单文件 100 MB，仓库建议控制在 1 GB 以内），jsDelivr 免费加速。
- **无外链限制**：只要遵守 GitHub 合理使用政策，图片链接不会像某些免费图床一样被替换成广告页。
- **可编程**：通过 GitHub REST API 或 `git` 命令即可完成图片上传，方便在 Python/Node.js 脚本中集成。
- **版本化**：利用 Git 标签或 commit hash 作为 jsDelivr 的版本号，可以有效管理图片更新，避免缓存穿透。
- **全球加速**：jsDelivr 拥有多 CDN 节点，在国外访问速度良好；国内部分地区可能较慢，但可作为内部工具的备选链路。

对于 OpenClaw 这类 Agent 平台，当插件需要返回图片 URL 时，一个固定的、受控的 CDN 链接远比临时生成的 base64 或本地文件路径可靠。

## 动手搭建：从零到 CDN 链接

### 1. 准备仓库
创建一个**公开的** GitHub 仓库，命名如 `static-assets`。不建议混合存放代码，保持仓库职责单一。

### 2. 上传图片
**手动方式**：直接拖拽文件到仓库的 Web 界面提交。  
**脚本方式**（推荐自动化）：
```bash
# 通过 git 操作
git clone https://github.com/YOUR_USER/static-assets.git
cp /path/to/screenshot.png static-assets/images/
git add images/screenshot.png
git commit -m "add screenshot"
git push
```
或使用 GitHub API 以 Base64 方式上传（适合无 git 环境的轻量脚本）：
```python
import requests, base64
with open('photo.png', 'rb') as f:
    content = base64.b64encode(f.read()).decode()
res = requests.put(
    'https://api.github.com/repos/YOUR_USER/static-assets/contents/images/photo.png',
    headers={'Authorization': 'token YOUR_TOKEN'},
    json={'message': 'upload','content': content}
)
```

### 3. 获取 jsDelivr 链接
标准格式：
```
https://cdn.jsdelivr.net/gh/用户名/仓库名@版本号/文件路径
```
- 不加 `@版本号` 则指向默认分支最新提交（有缓存延迟）。
- 推荐使用 `@master` 或具体的 commit hash，避免图片更新后旧链接仍访问旧缓存。

例如：`https://cdn.jsdelivr.net/gh/john/assets@a1b2c3d/screenshots/2025-04-01.png`

### 4. 在项目中使用
在 OpenClaw 插件或 MCP 工具中，直接返回该 CDN 链接。可以配合环境变量动态拼接基础 URL，方便切换不同仓库或缓存策略。

## 自动化实践：配合 GitHub Actions

为了让图片上传与 Agent 工作流深度集成，可以利用 GitHub Actions 实现**监听特定事件并自动拉取/上传图片**。例如：当 Issue 中贴出图片时自动转存到图床仓库。

一个简单场景：本地脚本生成截图后，通过 Action 触发上传。可创建一个 `upload.yml` 放在图床仓库中：
```yaml
name: Manual Upload
on:
  workflow_dispatch:
    inputs:
      filename:
        description: '图片文件名'
        required: true
      image_base64:
        description: 'Base64 编码的图片'
        required: true
jobs:
  upload:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Decode and save image
        run: |
          echo "${{ github.event.inputs.image_base64 }}" | base64 -d > images/${{ github.event.inputs.filename }}
      - name: Commit and push
        run: |
          git config user.name "github-actions"
          git config user.email "actions@github.com"
          git add images/
          git commit -m "upload ${{ github.event.inputs.filename }}" || echo "No changes"
          git push
```
随后在外部脚本中通过 `gh workflow run` 或直接调用 Dispatch API 触发。这种方式避免了在本地存储 GitHub Token，也更安全。

## 踩坑记录与排障

1. **jsDelivr 缓存更新延迟**  
   推送新图片后，若使用无版本号链接，可能需要等待 CDN 主动回源（最多 24 小时）。强制刷新方法：访问 `https://purge.jsdelivr.net/gh/用户名/仓库名@分支/文件路径` 来清除缓存，但需文件确实存在且已同步。建议生产环境带上 commit hash 或版本标签，做到精准更新。

2. **仓库体积与滥用风险**  
   大量高频次上传、存储超大量图片可能触发 GitHub 的滥用检测机制，导致仓库被限制。作为图床仅适合**个人项目、低频写入**的场景。务必在脚本中加入速率限制，不要当作公共图床对外开放上传。

3. **国内访问偶发不稳定**  
   jsDelivr 在国内使用 `fastly` 或 `cloudflare` 边缘节点，部分地区运营商可能解析异常。对于面向国内用户的 Agent，可采用备用链接（例如相同图片同时推送到阿里云 OSS 并做回退）。实测在北、上、广等地区延迟可接受，但不要抱太高期望。

4. **OpenClaw 插件的回退策略**  
   在插件返回图片前，可先请求一次 CDN 链接（HEAD 请求）验证可达性，若 5 秒内无响应则返回本地 base64 或占位图。这种防御性编程能提升工具的鲁棒性。

## 可复用建议

- **统一图片命名规则**：`YYYYMMDD-HHMMSS-描述.png`，避免覆盖和混乱。
- **启用 Git LFS**：如果单个图片容易超过 100 MB（如未经压缩的原图），建议提前开启 LFS，否则 GitHub 会拒绝推送。
- **使用环境变量管理基础 URL**：将 `https://cdn.jsdelivr.net/gh/user/repo` 放在配置中，方便迁移到其他 CDN（如换成自己的域名）。
- **配合 Image Processing 管道**：jsDelivr 本身不支持裁剪，可以在上传前通过 Pillow 或 sharp 压缩图片，减小体积的同时减少缓存回源时间。
- **监控仓库容量**：定期检查仓库大小，接近 1 GB 时清理旧图或归档至另一个仓库。

## 总结

GitHub + jsDelivr 搭建的免费图床，在可控范围内为 OpenClaw 的自动化实践提供了一种稳定、可编程的图片托管选择。它不是高并发场景的银弹，但对于原型开发、内部工具、个人知识库来说，完全够用。通过 Git 工作流结合 GitHub Actions，可以把图片的生成、上传、CDN 交付彻底自动化，使 Agent 的输出更加完整和可消费。

在工程实践中，最值得投入的是**缓存刷新策略**和**回退机制**，它们是免费方案能否可靠运行的基石。如果你也厌倦了随时可能失效的第三方图床，不妨试试这套组合，把图片托管纳入自己的代码仓库管理体系。

---

