---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 35754
source: 综合讨论
publishedAt: 2026-09-02
---

## 背景

给 Agent 接上 shell 和文件系统工具之后，第一反应是兴奋，第二反应是后怕：它拿到的是和普通人一样的 `rm -rf` 能力，而 LLM 的输出并不确定。OpenClaw 的设计假设很直接：**不要指望模型永远不犯错，而是假定它一定会犯错，然后用沙箱把错误的爆炸半径限制在可恢复范围内。**

这篇帖拆一下 OpenClaw sandbox 的四层模型，以及为什么"误删文件"在默认配置下基本不会发生。

## 问题：误删的真实来源

复盘社区案例，"Agent 删错东西"很少是模型"想删"，更多是这几类：

- 路径解析错误：相对路径 + 意外的 cwd，`rm -rf ./build` 落到了别处的 build；
- 上下文混淆：把 A 项目的清理习惯带到 B 项目，对错误目录执行了脚本；
- 幻觉路径：模型编造了一个"看起来合理"的绝对路径；
- 工具越权：某个自定义 MCP 工具本身没做任何限制。

靠 prompt 写"请小心删除"防不住这些。OpenClaw 的思路是把安全做在模型外面。

## 做法：四层防线

**第一层：进程降权。** 工具执行默认跑在低权限工作进程（或容器）里，对 `$HOME` 和系统目录没有写权限。模型就算输出最恶劣的命令，内核这一关也过不去。

**第二层：文件系统沙箱。** 所有 FS 工具的路径先做规范化（解析 `..` 和 symlink），再校验是否落在 workspace root 之内。allowlist 之外的读写被网关直接拒绝，模型只会收到一条结构化的 permission denied，然后自行调整策略。

**第三层：工具级门禁。** 路径合法还不够，网关按操作类型分级：读操作放行；写操作走 patch/diff 工具而非裸 shell；不可逆操作（rm、批量 mv、force push）必须人工确认或直接禁用。也就是说，模型没有一条"干净的路径"能直接调到 `rm`。

**第四层：审计与干跑。** 每次工具调用落审计日志（时间、工具、规范化后的路径、结果），关键命令支持先 dry-run。

配置上三步：

1. 显式声明 workspace root，一个项目一个目录，不要把整个 home 给出去；
2. 打开 sandbox 模式，保持默认 deny，按需加 allowlist；
3. 用金丝雀文件验证：让 agent 尝试删除 workspace 外的文件，确认被拦截，再翻审计日志。

## 踩坑点

- **symlink 逃逸**：早期实现只做字符串前缀匹配，`~/ws/link -> /etc` 这类链接能绕过检查。现在是先 resolve 再比对，但你自己写的 MCP 工具要记得同样处理。
- **自带 MCP 工具绕过沙箱**：沙箱只保护经过网关的工具。挂一个"直接 exec"的自定义 MCP server，等于把前三层全拆了。宁可让它复用内置 FS 工具，也别图方便开裸 shell。
- **root 下运行**：容器里图省事用 `--user root`，第二层防线直接失效。
- **确认疲劳**：门禁太松会天天弹确认框，人会机械地点"允许"，门禁形同虚设。宁可收紧 workspace 划分，不要放宽门禁。
- **Docker 挂载过宽**：把整个 home 挂进容器，容器内再严也守不住宿主机。

## 可复用建议

1. 最小权限是唯一的正解：workspace 按项目切，能用 FS 工具就不给 shell；
2. 上线前做一次"越狱演练"：让 agent 尝试访问 workspace 外路径，确认各层都拦得住；
3. 定期翻审计日志，重点看被 deny 的调用——那是最真实的模型行为画像；
4. 自定义工具一律默认 deny，路径先 canonicalize 再校验；
5. 把"误删"当可用性问题而非安全问题：有了 git + patch-first 写入，即使删错也在版本库里。

## 总结

"Agent 不会误删文件"不是一句承诺，而是一个架构结果：模型负责决策，沙箱负责兜底，人负责不可逆操作。OpenClaw 把信任边界放在网关而不是 prompt 里，这才是它敢让 agent 碰真实文件系统的原因。这套分层思路（sandbox root + 路径规范化 + 工具分级 + 审计）也完全可以搬进你自己的 agent 项目。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/e21a0d33c7ec01ef.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/2fb49d0675aa1793.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-02/c82212ca59814507.png)

