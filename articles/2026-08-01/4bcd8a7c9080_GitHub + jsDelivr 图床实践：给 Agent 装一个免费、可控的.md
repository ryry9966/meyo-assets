---
title: GitHub + jsDelivr 图床实践：给 Agent 装一个免费、可控的图片 CDN
feedId: 31175
source: 综合讨论
publishedAt: 2026-08-01
---

## 背景

在 OpenClaw、Agent、MCP 这类自动化工作流中，时常需要将生成的截图、数据可视化、OCR 结果等以图片形式输出，并返回一个可直接访问的 URL。无论是作为 Agent 间的数据传递，还是插入到报告、聊天卡片中，可靠、长久、无额外认证的图片托管都是刚需。

市面上的图床要么限速限容量，要么需要登录会话，要么 API 复杂且收费。对自建 MCP 工具或脚本来说，更希望有一个**免费、可控、版本化、能被 CDN 加速**的方案。GitHub 仓库 + jsDelivr CDN 恰好满足这些需求：图床本身是一个 Git 仓库，上传通过 GitHub API，链接被 jsDelivr 全球 CDN 加速，整体零成本，且天然适合自动化。

## 问题

选型时主要考虑四个点：

1. **稳定与速度**：国内用户访问部分免费图床极慢或被墙，jsDelivr 在国内设有节点，实测延迟可接受。
2. **权限控制**：只需个人 token，不必引入第三方服务端的权限模型。
3. **自动化集成**：能否通过简单 HTTP 请求完成上传与取链，便于封装成 MCP 工具或 Agent 的函数调用。
4. **运维成本**：无需维护额外服务器，仓库即存储。

GitHub + jsDelivr 将存储与分发解耦，仓库负责持久化和版本控制，CDN 负责加速，这种架构让图片管理像管理代码一样清晰。

## 做法 / 步骤

### 1. 创建图床仓库并获取 Token
在 GitHub 新建一个公开仓库，例如 `cdn-images`。进入 Settings → Developer settings → Personal access tokens (classic)，生成一个具有 `repo` 权限的 token，记下备用。

### 2. 手动上传一张图片理解流程
将图片 push 到仓库任意路径，例如 `test/hello.png`。其 raw 原始链接为：
```
https://raw.githubusercontent.com/<user>/<repo>/main/test/hello.png
```
对应 jsDelivr CDN 链接只需替换 host 和路径规则：
```
https://cdn.jsdelivr.net/gh/<user>/<repo>@main/test/hello.png
```
注意：CDN 链接必须指定分支（如 `@main`），且路径大小写敏感、不可省略文件名。该链接会触发 jsDelivr 拉取并缓存文件，全球生效通常在几秒内。

### 3. 自动化上传脚本（适合集成到 MCP/Agent）
用 Python 封装一个极简上传函数，接受文件路径或二进制数据，返回 CDN 链接。核心 API：`PUT /repos/{owner}/{repo}/contents/{path}`，需 Base64 编码文件内容。

```python
import requests, base64, os

def upload_to_cdn(file_path: str, repo="your/cdn-images", token=os.getenv("GITHUB_TOKEN")):
    with open(file_path, "rb") as f:
        content = base64.b64encode(f.read()).decode()
    # 文件名统一小写、替换空格
    fname = os.path.basename(file_path).lower().replace(" ", "-")
    api_url = f"https://api.github.com/repos/{repo}/contents/{fname}"
    headers = {"Authorization": f"token {token}", "Accept": "application/vnd.github.v3+json"}
    payload = {"message": f"upload {fname}", "content": content}
    resp = requests.put(api_url, json=payload, timeout=30)
    resp.raise_for_status()
    raw_url = resp.json()["content"]["download_url"]
    # raw URL 转 jsdelivr
    cdn_url = raw_url.replace("raw.githubusercontent.com", "cdn.jsdelivr.net/gh")
    cdn_url = "/".join(cdn_url.split("/")[:5]) + "/" + cdn_url.split("/")[-1]
    # 确保为 @分支 形式
    cdn_url = cdn_url.replace("/gh/", "/gh/").replace("/main/", "@main/")
    return cdn_url
```

在 MCP 工具定义中，将该函数暴露为一个 `upload_image` tool，接受文件路径参数，返回 CDN 地址。Agent 执行截图命令后直接调用此工具，便完成了图片持久化与可访问链接的生成。

### 4. 可选优化：每天一个子目录、定时清理
可在脚本中加入时间戳或日期作为目录前缀，避免单目录文件过多。GitHub 限制单仓库 1GB，非恶意使用几乎不会触达。定期清理过期图片可用一个 GitHub Actions 跑 cron，删除 N 天前的文件并提交。

## 踩坑点

1. **jsDelivr 缓存刷新**  
   如果覆盖同名文件（相同路径），jsDelivr 默认缓存 7 天左右，新内容不会立即生效。对策：**文件名带版本号或内容哈希**，每次上传都是新文件。如果确实需要覆盖，可手动请求 jsDelivr purge 接口（`https://purge.jsdelivr.net/gh/<user>/<repo>@<branch>/<file>`），该接口仅允许少量频率调用，适合非常规操作。

2. **国内访问抖动**  
   虽然 jsDelivr 在国内有节点，但 `cdn.jsdelivr.net` 域名偶尔被 DNS 污染或访问缓慢。备选域名 `fastly.jsdelivr.net`（走 Fastly CDN）或 `gcore.jsdelivr.net` 有时表现更好。可封装一个 URL 重试逻辑，检测不可用时切换子域。

3. **API 限流与并发**  
   未认证的 GitHub API 每小时仅 60 次请求。使用 token 后可达 5000 次/小时，对于常规 Agent 使用绰绰有余。如果出现 `422 Unprocessable Entity`，多半是文件已存在且未提供 `sha` 值（用于更新），此时选择自动追加版本号即可规避。

4. **文件名与路径**  
   jsDelivr 对大小写敏感，且路径中不能有特殊字符（空格、#等）。最佳实践：文件名一律小写，使用连字符，避免中文和特殊符号。

5. **Token 安全**  
   永远不要将 token 硬编码在代码或仓库中，需通过环境变量或 Secrets 管理。若在 GitHub Actions 中使用，直接引用 secrets。对于 Agent 本地运行，可将 token 放在 `~/.netrc` 或系统环境变量。

## 可复用建议

- **封装为 MCP 工具**：支持 `upload_file`、`list_files`、`delete_file` 三个端点，满足 Agent 对图床的完整操作需求。
- **统一接口隔离**：不直接暴露 GitHub 细节，内部自动处理文件名哈希、子目录、CDN 域名容灾，对 Agent 仅暴露 `provide_image_link` 这样的语义化接口。
- **配合截图场景**：如果你的 Agent 使用 Playwright 截图，可将 Base64 截图直接传入上传函数，减少写磁盘步骤。
- **监控与日志**：记录每次上传的文件大小和状态码，出现 401、403 及时告警（可能是 token 过期或权限变动）。

## 总结

GitHub + jsDelivr 组合提供了一种零成本、高可控的图片 CDN 方案，非常适合 Agent 与 MCP 工作流中的轻量级图片托管。它把“文件存储”与“全球加速”清晰分离，借助 Git 的版本控制，还能追溯每次图片变更历史。通过简单的 API 封装，你的 Agent 就拥有了一个永久在线、随时可访问的图床能力。

对于日上传量在百张以内、依赖个人项目或小规模自动化的场景，这套方案足够可靠且务实。如果需要服务大量并发用户，建议迁移到专业的对象存储 + CDN 服务，但作为“让跑起来的工具”，它已经做到了刚刚好。

---

