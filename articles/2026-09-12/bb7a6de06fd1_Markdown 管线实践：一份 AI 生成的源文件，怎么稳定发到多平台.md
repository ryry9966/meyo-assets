---
title: Markdown 管线实践：一份 AI 生成的源文件，怎么稳定发到多平台
feedId: 37187
source: 综合讨论
publishedAt: 2026-09-12
---

## 背景

在 OpenClaw 的自动化场景里，让 Agent 产出内容只是第一步。真实需求往往是“发布”这个动作：同一篇文章要进公众号、发知乎、投掘金，或者落到自建博客和 RSS。各平台接受的并不是同一种 Markdown，直接把 LLM 输出粘贴进去，格式大概率崩。

## 问题

拆开看是三层：

1. **方言差异**：GFM 表格、脚注、公式、任务列表，各平台支持度不一；公众号甚至不收 Markdown，只收内联样式的 HTML。
2. **资产问题**：外链图片多半被防盗链拦掉，公众号图片必须先传素材库换回自家链接。
3. **转换归属**：让 LLM 直接“转成公众号格式”，输出不确定，转义常错，而且没法做回归测试。

## 做法

原则一句话：**内容归模型，格式归代码。**

1. **约束源格式**。在给 Agent 的 prompt 里限定一个规范 Markdown 子集：仅二级标题起步、GFM 表格、标准代码块、图片用相对路径引用，禁止内联 HTML。配一份 markdownlint 规则当门禁。
2. **规范化**。发布流程先跑 lint，再把图片下载到本地 `assets/`，保证源文件自包含、可离线复现。
3. **适配器层**。每个平台一个 adapter，接口统一：

```python
class PlatformAdapter(Protocol):
    def transform(self, md: str, assets: AssetMap) -> Payload: ...
    def publish(self, payload: Payload, dry_run: bool) -> PublishResult: ...
```

公众号走 markdown-it 渲染 HTML 后用 inline CSS 方案压样式；知乎、掘金直接传 Markdown 但按各自限制裁剪语法；博客和 RSS 走静态站模板。
4. **资产管线**。adapter 内部调用平台素材 API（公众号的 access_token + 上传接口），上传后重写图片 URL。重写必须在 transform 内完成，不要事后全局字符串替换。
5. **幂等与回滚**。以 content hash + 平台 post id 建映射表，重跑管线不会重复发稿；某个平台失败不影响其他平台。
6. **验证**。dry-run 输出渲染快照，对每个转换器做 golden test，改样式时 diff 一眼看清。

在 OpenClaw 侧，把每个 adapter 包成一个 MCP tool 是比较自然的落点：Agent 只决策“发到哪”，格式细节全在 tool 实现里，模型不碰转义逻辑。

## 踩坑点

- **公众号的代码块最容易翻车**：代码里的 `<`、`&` 要经过两轮转义（先 HTML 实体，再过样式内联工具），漏一轮就丢字符，而且是概率性出现。
- **单换行语义不同**：部分渲染器把单换行当硬换行，CommonMark 不算，结果就是段落粘连或断行过度。最好在规范层强制空行分段，别指望下游修。
- **表格与公式**：知乎对 GFM 表格兼容但公式要 LaTeX 源码；公众号公式要么转图片要么放弃。在 lint 里直接禁止，比事后降级省事得多。
- **access_token 过期**：素材上传失败最常见的原因不是图片超限，而是 token 缓存没做刷新。
- **让 LLM 输出 HTML 是反模式**：一次成功不等于次次成功，转义错误测不住。

## 可复用建议

- adapter 当插件写，接口固定。新平台等于新插件，主管线零改动。
- 转换器不碰网络，资产上传独立成一层，方便 mock 和单测。
- 发布映射表用 SQLite 就够，别上重依赖。
- 上线“全自动群发”之前，先跑稳定“dry-run + 快照确认”，自动化程度可以慢慢加。

## 总结

这套管线的核心不是选哪个转换库，而是三个约束：**单一规范源、确定性适配器、幂等发布**。模型负责内容质量，代码负责格式正确性，职责切干净之后，新增平台只是多写几百行 adapter。OpenClaw 的插件机制很适合承载这类实现，欢迎在社区里共享各平台的适配经验。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/b67d0cb366e18d3d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/ceb9fac152c1cf92.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-12/f864bd7cba26232d.png)

