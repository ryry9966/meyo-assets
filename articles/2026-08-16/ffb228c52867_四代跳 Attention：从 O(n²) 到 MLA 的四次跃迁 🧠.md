---
title: 四代跳 Attention：从 O(n²) 到 MLA 的四次跃迁 🧠
feedId: 01M05FSWNP8AVKCB71608593KN
source: weibo
publishedAt: 2026-08-16
---

微博上“四代跳 Attention”突然被顶上来，不少同学第一反应是：Attention 不就是 softmax 乘一下 V 吗，怎么能跳出四代？如果只看数学定义，确实只有一代；但把 Attention 放进大模型训练和推理的真实显存/带宽账本里，它至少已经被工程化地“跳”了四次。今天就把这四次跃迁拆开看。



## 先给一张代际地图

- 第0代：Softmax Attention——标准答案，O(n²) 计算与显存
- 第一代：Sparse Attention——不全看，只挑重点 token
- 第二代：Linear Attention——先算 K^T V，绕开显式 n×n
- 第三代：FlashAttention——I/O 感知，不物化完整注意力矩阵
- 第四代：KV Cache 瘦身——GQA / MQA / MLA 为推理减负

这是为了便于理解做的技术化约，不是论文官方代际。真正重要的不是记住“第几代”，而是每一代到底解决了哪张账本上的问题。

## 第0代：Softmax Attention 是基本盘

标准公式：

`Attention(Q, K, V) = softmax(QK^T / √d_k)V`

对序列长度 n，QK^T 会形成一个 n×n 矩阵。计算复杂度是 O(n²d)，显存占用也是 O(n²)。这意味着：

- 2048 token 的序列，每个 head 要算约 419 万个标量。
- FP32 下，一个 n×n 矩阵约 16 MB；模型一层有多个 head，叠上 batch 和层数，显存会迅速爆炸。
- 长文档、代码库、视频帧一旦要读 128k token，平方增长直接不可训。

第0代不是不好，而是太老实：每个 token 都要跟所有 token 算一遍亲密度。当序列变长，它的“全连接”方式就会成为工程瓶颈。

## 第一代：稀疏化——从“全都要”到“只看重点”

直觉来自语言本身的局部性：不是每个词都需要看所有词。

- Longformer：sliding window + global tokens，局部窗口内做 attention，少数全局 token 负责全局信息。
- BigBird：random + window + global 三种注意力组合。
- Reformer：用 LSH 把相似 query/key 哈希到同一桶，只对桶内 token 计算。

复杂度可以降到 O(n·w·d) 或 O(n log n)，w 是窗口大小。代价是：

- 窗口外的全局信息可能丢。
- global token 需要任务先验，不是万能超参。
- 不同任务通常要重新调窗口和稀疏模式。

这一代适合长文档分类、检索增强等场景。它不是数学上更美，而是算得更省。

## 第二代：线性化——把 softmax 拆掉

既然 softmax 逼着我们先算完整的 QK^T，那就把 softmax 拆掉或用核函数近似。

- Performer 用随机特征核 φ(q)φ(k) 近似 softmax，利用结合律先算 K^T V，复杂度降到 O(n d²)。
- Linformer 假设注意力矩阵低秩，将 K、V 投影到固定长度 k，复杂度 O(n k d)。

当 d 远小于 n 时，这一代显著降低计算量。代价也很明确：

- 近似误差可能让长尾 token 被忽略。
- 训练和推理时对算子、数值精度要求更敏感。
- 工程实现不如标准 softmax 稳定。

所以这一代更多是研究推动和方法储备，不是大一统替代。

## 第三代：FlashAttention——I/O 感知的分水岭



FlashAttention 不改变 attention 公式，它改的是计算图落地方式。

普通实现里，GPU 会先算 S = QK^T，写回 HBM；再读出来做 softmax；再写回；再算 PV。HBM 带宽成为真正的瓶颈。FlashAttention 的做法是：

- 把 Q、K、V 切成小块，塞进 SRAM。
- 用 online softmax 维护 running max 和 running exp，分块计算。
- 不把完整 n×n 注意力矩阵写回显存。
- 反向时重计算中间结果，而不是保存大矩阵。

计算量仍然是 O(n²)，但实际速度大幅提升，显存占用从 O(n²) 降到 O(n)。类比一下：不是改了菜谱，是改了后厨流水线——少跑仓库，尽量在案板上切完。

FlashAttention-2 继续减少非矩阵乘操作，提高 GPU 占用率；FlashAttention-3 则利用 Hopper 的 TMA/WGMMA 做异步和低精度计算。现在训练长上下文模型，FlashAttention-2/3 基本是默认选项。

## 第四代：KV Cache 瘦身——GQA / MQA / MLA

背景是推理阶段。Decoder 每生成一个 token，都要读一次 KV cache。原始 Multi-Head Attention 每个 head 都有一套 K、V，显存大，带宽高度敏感。

于是出现三类结构优化：

- MQA：所有 query head 共享一套 KV，最省但可能伤效果。
- GQA：多个 query head 分到一组共享 KV，在省显存和保持效果之间折中。
- MLA：把 K、V 低秩联合压缩到 latent space，推理缓存更小，注意力计算前再解压，进一步压缩 KV cache。

PagedAttention 是系统层优化，主要解决 KV cache 的碎片化和动态管理。第四代本质不是减少 FLOPs，而是降低推理单位 token 的显存与带宽成本。

## 冷思考：别只看名字，要看账本



追“四代跳 Attention”这类热点时，有三笔账一定要先算：

- 先判断瓶颈是 compute 还是 memory bandwidth。短序列、小模型下 FlashAttention 收益可能不明显；长序列才会拉开差距。
- 稀疏/线性 attention 有近似代价，先测长文本 recall、QA 等真实任务，不要只看训练 loss 掉没掉。
- 部署时算三笔账：KV cache 显存、prefill/decode 的带宽、batch size。再决定上 GQA / MLA / PagedAttention，不要盲目追新名词。

## 实用建议

1. **学习路径**：先手推 O(n²) 的 Softmax Attention，再画 FlashAttention 的分块图，比背论文标题更划算。
2. **工程选型**：训练长上下文优先 FlashAttention-2/3；推理低显存优先 GQA/MLA；服务层叠加 PagedAttention 与量化。
3. **追新名词时问三句**：它减少了什么？数学等价还是近似？对现有框架和硬件是否友好？答不上来就先不引入。

Attention 还在跳，但脑子别跟着跳。抓住计算账、显存账、I/O 账，大部分“新 attention”都能被拆成这三张账本。
