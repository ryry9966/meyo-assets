---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 35864
source: 综合讨论
publishedAt: 2026-09-03
---

## 背景

OpenClaw 的典型用法是让 Agent 长期驻留，通过 shell / 文件工具直接操作真实工作区：批量重构、清理产物、跑自动化任务。也就是说，Agent 手里天然握着一份「能对文件系统执行写操作」的权限。模型越能干，单次误操作的期望损失就越高。

## 问题

「Agent 会不会哪天把我 home 目录删了」——这不是段子，可触发路径至少有四类：

- **路径幻觉**：模型把 `~/workspace` 解析成 `/workspace`，或拼接出 `../../`；
- **指令泛化**：「清理一下构建产物」被理解成删除范围过大的 glob；
- **提示注入**：Agent 拉取的外部 README、网页内容里夹带破坏性命令；
- **自动化失稳**：定时任务在异常状态下重试，反复执行删除逻辑。

要明确一点：靠 system prompt 写「请不要删文件」不构成安全边界。模型偶尔一定会犯错，真正的问题是——错误发生时，爆炸半径有多大。

## 做法：纵深防御的四层

OpenClaw 的思路不是「让模型变靠谱」，而是假设它必然犯错，用环境兜底：

1. **沙箱隔离**。工具执行默认落在 Docker 容器里，宿主机只 bind-mount 一个 workspace 目录（读写），其余文件系统对 Agent 不可见。误删的物理上限就是这个目录。
2. **最小权限**。容器内以非 root 用户运行，按需裁剪 capabilities；网络默认关闭或走白名单，降低注入成功后外联的可行性。
3. **路径围栏**。文件类工具执行前对路径做 canonicalize，解析符号链接后校验是否仍在 workspace 内，`..` 穿越和软链逃逸直接拒绝。
4. **破坏性命令闸门**。识别 `rm -rf`、`find -delete`、`mkfs`、批量覆盖写等模式，命中后升级为人工确认，其余正常放行；工具调用全量落审计日志。

配置上大致三步：

```yaml
sandbox:
  enabled: true
  workspace: ~/openclaw-workspace   # 唯一读写挂载
  network: deny
  confirm_on_destructive: true
```

建议先在一个 scratch 目录里故意下「删掉所有临时文件」的指令，观察围栏和闸门是否真的触发，再接入真实工作区。

## 踩坑点

- **挂载粒度错误**：把 `$HOME` 或上级目录挂进容器，围栏形同虚设。workspace 必须是独立的最小目录。
- **软链陷阱**：workspace 里一个指向 `~/` 的 symlink 就可能造成逃逸，校验必须在 resolve 之后做。
- **确认疲劳**：闸门阈值太松，用户逢弹窗就点允许，安全闸退化成仪式。宁可收紧模式列表，也别让确认变得廉价。
- **容器内写丢**：Agent 往容器内层文件系统写文件、没落在挂载卷上，重启后「文件消失」，看起来像误删。检查写路径是否都在挂载点内。
- **定时任务旁路**：cron 直接调宿主机脚本，绕过了沙箱。所有自动化入口都应走同一套沙箱配置。

## 可复用建议

- 把「爆炸半径」当设计指标：任何新工具接入前先回答一句，它最坏能影响哪些路径。
- 批量操作前让 Agent 先打 git commit。回滚比拦截更可靠，两者都要有。
- 审计日志定期人工抽检，比事后追责有用得多。
- 用对抗性 prompt 定期演练沙箱。配置漂移比漏洞更常见。

## 总结

Agent 不误删文件，从来不是因为模型足够聪明，而是因为环境让它删不到。OpenClaw sandbox 的价值在于把「模型会犯错」当成默认假设：隔离限定损失上限，围栏和闸门抬高犯错成本，git 和审计提供兜底。Prompt 可以引导行为，但只有环境约束才是真正的安全边界。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/9805f3fc17c017de.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/8301600be89ded3e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-09-03/bff17fef66e24e2d.png)

