---
title: Token怎么造？黄仁勋五层蛋糕拆解⚙️
feedId: 01M0XYM8192N3PT3J90K7NHYFC
source: 36kr
publishedAt: 2026-08-26
---

为什么同一个问题，有的模型按字收费，有的按 token 收费？一个 token 既不是一个汉字，也不是一个英文单词，而是经过一条工业链路“制造”出来的最小语义单位。NVIDIA 黄仁勋曾把 AI 基础设施比作五层蛋糕：数据、Token 化、Embedding、Transformer 和推理采样。这个说法未必严谨到论文级别，但用来理解 token 的诞生过程非常直观。下面顺着蛋糕层拆一遍。

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/de248be8f1214eda.png)

## 第一层：数据清洗——先把“小麦”筛干净

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/cabcce15ccbaca8f.png)

- 原始数据来自网页、代码仓库、电子书、学术论文、维基百科和多语言语料。它们像刚收上来的小麦，杂质很多，不能直接进模型。
- 清洗步骤包括去重、丢弃低质量文本、过滤隐私与敏感内容、调整语言配比。很多机构还会对代码和数学内容做专门筛选。常用手段包括精确去重、MinHash 近似去重、质量打分器和困惑度过滤。
- 数据配比同样重要。如果全是网页语料，模型会偏口语化；代码和书籍有助于推理与长文能力，因此需要刻意平衡。
- 预训练数据量通常远大于最终 token 数。例如 Meta 的 Llama 3 公布 15T tokens 训练量，原始抓取规模可能是数倍，大量内容在清洗阶段被扔掉。
- 这一层决定 token 的“出身”。低质量数据会让后续 token 携带错误先验，模型能力再强也难完全纠正。

## 第二层：Tokenization——切出AI认识的“乐高块”

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/44ea1a73e6d618b0.png)

- 人类靠词和字理解句子，模型需要稳定、可逆、覆盖面广的切分规则。主流方案是 BPE（Byte Pair Encoding）及其变体 SentencePiece、WordPiece。
- BPE 把高频子词逐步合并成 token：既保留常见词，又能处理生僻词。举例说，"unbreakable" 可能被切成 "un" + "break" + "able"；中文则常按字或常见双字组合切分。
- 词表大小差异很大：GPT-4 词表约 100k，有些模型 32k，有些 200k。词表大，同样的句子需要的 token 更少，但 embedding 层也更占显存。
- 除了普通文本 token，还会有 BOS、EOS、PAD、UNK 等特殊 token，用来标记句子边界和未知内容。
- Tokenization 直接影响成本和中英文体验。切太粗，含义模糊；切太细，序列变长，计算更贵。

## 第三层：Embedding——给Token一个高维“门牌号”

![img3](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/eec2fefa909cee1d.png)

- token 进入模型前，会被映射成向量。常见维度有 768、1024、4096 甚至更高，这是给矩阵计算用的，不是给人看的。
- Embedding 是上下文相关的：同一个 "bank"，在 "river bank" 和 "central bank" 中会得到不同向量，因此它不是死词典，而是动态语义表示。
- Embedding 矩阵和 Transformer 参数一起训练。经典案例是 king - man + woman ≈ queen，说明向量空间能编码语义关系。
- 工程上，词表大小 × 向量维度就是 embedding 矩阵的参数量。例如 100k 词表、4096 维，仅输入 embedding 就约 4 亿参数。
- 维度越高表达上限越高，但显存和计算量也随之增加。模型设计师需要在表达力与效率之间找平衡。

## 第四层：Transformer与采样——在概率里“生”出下一个Token

![img4](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets@main/images/2026-08-26/80b87fdd75830394.png)

- 输入 token 经过多层 Transformer，每层用 self-attention 参考前文所有 token，更新当前表示。最后输出一个 logits 向量，长度等于词表大小。
- Softmax 把 logits 转成概率分布。模型生成时不是简单选最高分，而是按 temperature、top-k、top-p 等策略采样。这也是同一句提示词能产生不同答案的原因。
- 计算特性：每生成一个新 token，都要完整跑一遍前向。随序列变长，注意力计算近似 O(n²)。KV cache 能减少部分重复计算，但显存占用会上升。
- 黄仁勋“五层蛋糕”的最后两层——Transformer 计算和推理采样，其实共享同一次前向。前一环节产出概率，后一环节决定具体吐出哪个 token。
- 所以 token 单价不是“存一个词”的费用，而是为整条矩阵运算链付费。

### 冷思考：三条实用建议

1. **把 token 当数字物料，而不是字数。** 同一个中文问题，有的模型消耗 200 个 token，有的可能 350 个。预算要按 token 吞吐和单价算，不能只看汉字数量。
2. **提示词去冗余。** 空泛的背景描述、重复礼貌用语都会增加输入 token。长文档优先用检索、摘要或分段处理，而不是全部塞进上下文。
3. **模型选型看“软指标”。** 词表大小、中文分词质量、Embedding 维度、采样策略以及长上下文效率，往往比单纯的总参数量更影响实际体验。

token 不是凭空出现的，它是数据、切分、向量、矩阵和概率共同作用的产物。看懂这五层，再看 API 账单会顺眼很多。
