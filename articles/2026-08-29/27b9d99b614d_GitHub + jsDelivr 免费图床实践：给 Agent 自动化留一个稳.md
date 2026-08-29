---
title: GitHub + jsDelivr 免费图床实践：给 Agent 自动化留一个稳定图片出口
feedId: 35222
source: 综合讨论
publishedAt: 2026-08-29
---

## 背景

在 OpenClaw、Agent、MCP 插件和自动化流程里，图片经常不是最终目的，而是中间产物：插件截图、知识卡片、运行预览图、流程回调里的辅助图。它们需要变成一个稳定、可预测、可脚本化生成的 URL，再交给 LLM 阅读或写回 Markdown。

如果直接用临时上传服务，链接可能过期；用某些免费图床接口，容易被防盗链、限额、内容审核影响；用对象存储，则要维护密钥、计费和生命周期策略。对开源项目、个人自动化、小流量插件来说，GitHub 仓库 + jsDelivr CDN 是一个相对务实的选择。

它适合公开、非敏感的图片，不适合私密数据、大文件和高可用生产场景。

## 问题

选图片 CDN 时，真正影响自动化体验的通常不是“快不快”，而是这几点：

- URL 是否可预测，能不能按规则生成
- 文件覆盖后，CDN 是否会及时更新
- 上传能不能通过脚本或 API 完成
- 权限是否足够简单，且不会把敏感 Key 暴露在插件里
- 是否方便做版本控制和批量清理

GitHub + jsDelivr 的特点是：仓库公开、路径固定、可用 GitHub API 上传、jsDelivr 免费 CDN 加速，但缓存策略和文件限制需要提前搞清楚。

## 做法 / 步骤

### 1. 创建公开仓库

建议单独建一个 `assets` 或 `images` 仓库，不要混在业务代码仓库里。公开仓库里的所有图片都会对外可访问，先默认“任何上传到这里的内容最终都会公开”。

```bash
gh repo create my-assets --public --source=. --push
```

### 2. 设计目录结构

按业务或时间分目录，避免几万张图全部堆在根目录：

```text
/blog/2025/01/demo.png
/agent/run-12345/screenshot.png
/cards/mcp-intro.png
```

语义化路径比纯哈希好维护，但文件名建议加短哈希或时间戳，避免覆盖。

### 3. 上传图片

网页拖拽只适合手动场景。自动化和插件更适合用 GitHub API 提交文件。Python 伪代码大致如下：

```python
import base64
import requests

def upload_image(owner, repo, branch, path, file_path, token):
    with open(file_path, "rb") as f:
        content = base64.b64encode(f.read()).decode()

    url = f"https://api.github.com/repos/{owner}/{repo}/contents/{path}"
    headers = {
        "Authorization": f"Bearer {token}",
        "Accept": "application/vnd.github+json",
    }
    payload = {
        "message": f"upload {path}",
        "content": content,
        "branch": branch,
    }
    r = requests.put(url, headers=headers, json=payload)
    r.raise_for_status()
    return r.json()["content"]["download_url"]
```

建议使用 Fine-grained PAT，只给目标仓库的 Contents 写权限，不要把主力账号 Token 直接写进插件配置。

### 4. 生成 jsDelivr URL

上传到 GitHub 后，CDN 地址规则如下：

```text
https://cdn.jsdelivr.net/gh/{owner}/{repo}@{branch}/{path}
```

例如：

```text
https://cdn.jsdelivr.net/gh/yourname/my-assets@main/agent/run-12345/screenshot.png
```

分支可以用 `main`，但更推荐用 tag 固定版本，例如 `@v1.0.0`。这样后续即使分支内容变化，旧链接仍然指向固定版本，更适合归档场景。

### 5. 验证

```bash
curl -I "https://cdn.jsdelivr.net/gh/yourname/my-assets@main/agent/run-12345/screenshot.png"
```

关注是否返回 `200`，以及 `content-type` 是否为 `image/png`。首次请求可能触发 CDN 回源，稍慢是正常的。

## 踩坑点

### 1. 同名覆盖不会立即生效

jsDelivr 对同一 URL 的缓存可能比较持久。更新同名文件后，旧 URL 可能仍然返回旧内容。不要依赖“覆盖后自动刷新”。

正确做法是：

- 文件名带内容哈希，例如 `demo-3f2a1c.png`
- 或者每次发布使用新的 tag 或路径
- 需要更新时直接生成新 URL，而不是原地覆盖

### 2. 文件大小和仓库体积

GitHub 单文件建议控制在 20MB 以内，API 上传限制也受仓库策略影响。不要把这个方案当成大文件备份或视频存储。图片先压缩，再上传。

### 3. 公开性

图床仓库必须公开，否则 jsDelivr 无法访问。内部截图、带 token 的调试信息、用户隐私图片不要传。

### 4. API 限流

未认证 GitHub API 每小时只有 60 次请求。插件高频上传时，未认证脚本很容易触发限流。使用 PAT 或 GitHub App token 后，限额提高到 5000 次/小时，但仍要控制频率。

### 5. 大陆网络可用性

jsDelivr 在国内部分网络环境可能不稳定。小流量、个人自动化可以接受，但如果你的 Agent 插件被很多人使用，建议准备一个回退 URL 或镜像源，不要只依赖单一 CDN。

## 可复用建议

可以把上传逻辑封装成一个 MCP 工具或普通函数：

- 输入：本地文件路径、目标目录、可选分支
- 自动计算内容哈希并重命名
- 通过 GitHub API 上传
- 返回 `https://cdn.jsdelivr.net/gh/...` 格式的最终 URL
- 同时保存一份 `manifest.json`，记录本地文件与远端路径的映射

这样 OpenClaw 插件只需调用一个工具，不用关心 CDN、缓存和路径拼接。

另外，不要把所有图片都塞进同一个仓库。随着文件增长，克隆和批量操作会变慢。可以按年度、项目或用途拆仓库，或者定期归档旧文件。

## 总结

GitHub + jsDelivr 不是最强大的图床方案，但对公开图片、插件预览图、知识卡片和小型自动化来说，它的成本低、路径可预测、API 可脚本化，足够把“图片从生成到可访问”这一环做成稳定流程。

真正需要花心思的不是上传本身，而是：

- 文件名不能随意覆盖
- 目录要提前规划
- Token 权限要最小化
- 缓存策略要符合更新频率
- 隐私数据要明确隔离

免费不意味着零维护。对 Agent 自动化来说，可控路径、版本管理和脚本化上传，比“免费”本身更有价值。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/872e3f5095299d8c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/b367e2ed1433c954.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-29/d8bdbce8d7f92070.png)

