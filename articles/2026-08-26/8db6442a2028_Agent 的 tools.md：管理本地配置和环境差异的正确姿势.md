---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 34783
source: 综合讨论
publishedAt: 2026-08-26
---

## 背景

在 OpenClaw、MCP 插件和自动化 Agent 的实践里，工具定义往往散落在三处：代码里的 schema、提示词里的能力描述、运行机器上的实际命令/路径。工具的“元信息”和“运行环境”分离后，最常见的现象是：同一份 agent 配置，在 A 机器能正常调用 `ffmpeg`，换到 B 机器因为路径不同、环境变量缺失，Agent 反复失败或幻觉出错误命令。

tools.md 可以承担一个更明确的角色：它不是简单罗列工具名，而是把“工具能做什么”和“在当前环境怎么跑起来”合并成一份可读、可版本化、可被 Agent 引用的接口契约。

## 问题

实践中，多数 tools.md 只写工具名称、参数和一句话描述。环境信息要么完全缺失，要么被写死在正文里，比如：

- 直接写 `/home/me/bin/ffmpeg`，换机器后 Agent 继续按这个路径调用；
- 只写“需要 ffmpeg”，不提安装方式、版本差异、Windows/Linux 命令区别；
- 环境变量名、默认值、必填项不明确，Agent 调用时自己编一个 `API_KEY`；
- 本地能跑但 CI/容器里缺少系统依赖，Agent 无法判断失败原因。

这些问题的本质是：Agent 只拿到了工具的“业务接口”，没有拿到“运行时约束”。当 Agent 被要求处理本地文件、调用外部命令、读写特定目录时，环境差异会直接变成执行错误。

## 做法

建议把 tools.md 按“通用契约 + 本地覆盖”组织，而不是一个又大又全的混合文件。

### 1. 为每个工具补全运行时字段

至少包含：

- 工具名和用途；
- 输入/输出参数；
- 环境变量；
- 默认路径或查找规则；
- 平台差异；
- 失败时如何验证。

例如：

```markdown
## tool: transcode
- purpose: 转码音频
- env: FFMPEG_BIN, DATA_DIR
- default_bin: 从 PATH 查找 ffmpeg
- verify: `ffmpeg -version`
- platform: Linux/macOS 使用 `ffmpeg`，Windows 可能为 `ffmpeg.exe`
```

### 2. 环境差异只引用变量，不写死值

主 tools.md 里只写变量名和默认策略，不写本机绝对路径。真实值放到 `.env.local`、`config.local` 或部署环境变量中。这样 Agent 读到的是稳定契约，而不是某个人的机器路径。

### 3. 增加本地覆盖文件

可以在版本控制里保留 `tools.md` 作为通用契约，另建 `tools.local.md` 记录当前机器的特殊路径、代理地址、可用工具开关，并将其加入 `.gitignore`。Agent 读取顺序可以固定为：先读通用契约，再读本地覆盖；冲突时本地覆盖优先。

### 4. 让 Agent 在调用前做验证

在 system prompt 或项目说明中约定：凡是 tools.md 中标注了 `verify` 的工具，调用前先执行验证命令。如果验证失败，优先报告缺失条件，而不是继续尝试调用。这个规则可以显著减少“命令不存在”之类的低级错误。

### 5. 将 tools.md 纳入版本化和 CI 校验

将 tools.md 作为仓库的一部分，保持与真实工具 schema 同步。可以写一个简单的 smoke test：让 Agent 或脚本逐条读取 tools.md 中的 verify 命令并执行，失败项直接暴露。

## 踩坑点

1. **硬编码路径**：绝对路径是最容易踩的坑，尤其是数据目录、缓存目录、ffmpeg/node/python 路径。应一律使用环境变量或 PATH 查找。
2. **只写参数不写依赖**：Agent 知道要传什么参数，但不知道需要先 `source .venv/bin/activate` 或安装某个系统包。依赖和激活命令要写进去。
3. **tools.md 与真实 schema 不一致**：工具名或参数名在代码中改了，tools.md 没同步，Agent 会按旧信息生成错误调用。建议在 CI 里做 schema 对比或生成检查。
4. **把密钥写进 tools.md 并提交**：本地调试时方便，但容易泄露。密钥只出现在环境变量或本地未跟踪文件。
5. **平台差异被忽略**：Windows 和 Unix 的路径分隔、可执行文件后缀、命令别名都不同。至少写明“Linux/macOS”和“Windows”两列。
6. **环境探测被 Agent 忽略**：Markdown 不会被自动严格执行，需要在 prompt 中反复强调读取顺序和验证规则，否则 Agent 可能直接跳到工具调用。

## 可复用建议

- 保持 tools.md 只写“契约”，不写“实现细节”；实现细节交给脚本或本地覆盖。
- 将环境差异收敛为一张环境变量表：变量名、默认值、是否必填、示例值。
- 对高风险工具（涉及文件删除、外部命令、网络请求）增加“前置检查”和“失败兜底”字段。
- 在项目 README 或 Agent 全局说明中固定一段：

> 调用工具前先查看 tools.md 中对应工具的 Runtime requirements；如果存在 verify 命令，先执行 verify；不确定的路径不要猜测。

- 定期清理：删除不再使用的工具条目，避免 Agent 被过期信息干扰。

## 总结

tools.md 对 Agent 的价值不在于写得全，而在于把环境差异显性化、变量化和可验证化。一个工程上可用的 tools.md，应该让 Agent 在新机器上读完就能判断“这个工具能不能用、缺什么、怎么补”，而不是继续带着旧机器的路径和假设运行。先把这份接口契约维护好，比堆更多工具描述更实际。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/ff22e65aa85343a1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/11016738a219f056.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/af7b443613c61dac.png)

