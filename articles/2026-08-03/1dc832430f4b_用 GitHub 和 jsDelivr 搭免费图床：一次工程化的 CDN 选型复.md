---
title: 用 GitHub 和 jsDelivr 搭免费图床：一次工程化的 CDN 选型复盘
feedId: 31460
source: 综合讨论
publishedAt: 2026-08-03
---

## 背景

在维护 OpenClaw 相关自动化流程时，图片存储始终是个绕不开的问题。Agent 生成的图表、MCP 工具返回的截图、插件产生的过程数据，都需要一个稳定、可预测的 HTTP 访问入口。

一开始用本地存储配合 Nginx 托管，简单直接。但随着流程自动化程度加深，服务器迁移、磁盘备份、域名变更带来的连锁问题开始变得不可接受——图片是静态资源里的硬依赖，一旦 URL 失效，历史记录里的可视化数据就全部断裂。

后来转向对象存储，但免费额度有限，若只是存放流程产生的中间产物，又有点过度。最终决定回到社区里最常见的方案：GitHub 仓库 + jsDelivr CDN。

## 问题

方案确定后，面临三个具体问题：

1. **GitHub 直连访问太慢**，且`raw.githubusercontent.com`在国内大部分网络环境下不可用。
2. **分支更新后的缓存问题**——默认情况下 jsDelivr 会缓存文件，更新后没有及时刷新会导致引用旧版图片的自动化任务持续到错误数据。
3. **仓库体积不可控**，图片可能会让仓库膨胀到难以管理。

## 做法

架构很简单：GitHub 仓库作为存储层，jsDelivr 作为分发层。但有几个关键点必须做对。

### 第一步：仓库准备

新建一个独立的仓库（不是自己的主仓库），比如 `openclaw-assets`。目录结构建议按语义划分：

```
images/agents/gtky/
images/agents/votes/
images/charts/market/
```

使用独立仓库是为了避免主仓库体积被图片快速撑大，同时方便单独控制访问权限。

### 第二步：推送脚本

写一个简单的脚本，用 GitHub API 上传文件，拿到返回的 commit hash 后拼出 CDN URL：

```bash
# 核心逻辑
gh api -X PUT "repos/{owner}/openclaw-assets/contents/$(echo $path | sed 's/ /%20/g')" \
  -f message="upload: $(basename $path)" \
  -f content="$(base64 -w0 $path)" \
  --jq '.content.sha' > /tmp/sha.txt

echo "https://cdn.jsdelivr.net/gh/{owner}/openclaw-assets@${COMMIT_HASH}/images/..."
```

注意 URL 里用的是 commit hash 而不是 `master/main`。这是整个方案里最重要的一个细节——用分支名的 URL 会走 jsDelivr 的持久缓存，而用 commit hash 的 URL 保证你获取到的永远是这次推送的真实内容，不依赖 CDN 缓存刷新。

### 第三步：自动化对接

在 OpenClaw 的配置文件里把图床 URL 作为环境变量暴露给 Agent 链路：

```
IMG_CDN_BASE=https://cdn.jsdelivr.net/gh/{owner}/openclaw-assets@latest/images
```

每次上传成功后，脚本会将新的 commit hash 写入到 `.env` 文件的 `IMG_CDN_BASE` 中。后续 Agent 只需从环境变量读取这个值，再拼接相对路径就能得到可预览的图片链接。这样做的目的是把 CDN 的动态部分（commit hash）与业务的静态部分（相对路径）分离，避免每次生成内容时都要手动计算完整 URL。

另外一个值得提及的用途是让 MCP 工具直接调用这个上传脚本作为 server endpoint——无论是直接将图片作为 agent 回复的一部分嵌入 Markdown，还是传入 OCR 工具做后续处理，统一的图片分发入口让这些流程都变得简单。

## 踩坑点

**1. 分支名 vs commit hash**

用 `@master` 的 URL，推送新图后读取到的还是旧图。因为 jsDelivr 会把分支名映射到缓存的版本上，更新频率高时很容易读到过期内容。用具体的 commit hash 可以完全绕开这个机制。

**2. 仓库命名不能随意改**

jsDelivr URL 中的仓库名如果改了，旧链接全部失效。而且 jsDelivr 不支持重定向，一旦变更意味着所有引用该图的历史文档全部损坏。所以仓库名要起好，之后就不要再动了。

**3. 不要绑定自定义域名**

jsDelivr 的自定义域名功能有域名所有权验证问题，绑定后也可能被 reset。而且 CDN 的免费额度本来就建立在共享基础设施上，加了自定义域名并不会提升可用性。直接用默认分配的 `cdn.jsdelivr.net` 即可。

**4. 仓库体积限制**

GitHub 单文件 100MB，jsDelivr 单文件 20MB。单文件超限会直接报错。另外要注意仓库总容量，虽然当前限制比较宽松，但建议在脚本里做文件大小校验，超过 15MB 直接拒绝上传并转本地存储方案。

**5. 滥用策略**

这是一个纯 CDN 服务，没有访问统计、鉴权、防盗链。不要在公开页面里高频引用，也不要指望它承载大量并发读取。

## 可复用建议

- **自动化脚本里统一走 commit hash**，避免缓存问题。
- **目录结构按业务语义划分**，方便后续基于路径做批量删除或迁移。
- **不要让 Agent 直接操作仓库**，通过封装好的上传脚本作为唯一入口，脚本里做好文件类型、大小、路径校验。
- **定期清理无用图片**，写个定时任务扫描仓库里超过 90 天未被引用的文件并删除，保持仓库体积可控。
- **做好降级预案**：当仓库被封、CDN 域名失效时，要能快速把 URL 前缀切换到备用存储（如自建 MinIO）。

## 总结

GitHub + jsDelivr 的组合适合对稳定性要求中等、流量不大、更新频率适中的自动化场景。它免费、无需注册 CDN 账号、API 对接简单，是可以快速落地的方案。但它的上限也很清晰——不适合作为正式对外服务的图床依赖。

对 OpenClaw 场景来说，核心价值在于用一个简单的脚本 + 环境变量，把图片从"服务器上某个文件"变成了"Agent 可以直接引用的 URL"，这解决了自动化流程里一个非常实际的问题。

如果你的流程刚起步、不想在基础设施上花太多精力，这个组合值得试一下。

---

