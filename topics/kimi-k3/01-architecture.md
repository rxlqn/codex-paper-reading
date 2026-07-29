# 专题 1：KDA、AttnRes 与 Stable LatentMoE

## 核心问题

K3 的架构不是把普通 Transformer 做宽，而是分别处理三种信息流瓶颈：

| 维度 | 模块 | 要解决的问题 |
|---|---|---|
| Sequence / token | Hybrid KDA + Gated MLA | 百万 token 下既要线性效率，也要保留全局内容寻址 |
| Depth / layer | AttnRes | 深层网络不能只靠逐层 residual 累积信息 |
| Width / channel | Stable LatentMoE | 896 experts 的极稀疏路由必须稳定且负载均衡 |

这三部分共同构成 K3 所谓的“scaling information flow”，比只看参数量更接近架构贡献。

## 1. Hybrid Attention

一个 block 包含 3 个 KDA 层和 1 个 Gated MLA 层，93 层 backbone 合计 69 KDA + 24 MLA，并在末层保证 global attention。

### KDA 的作用

KDA 用固定大小的 recurrent state `S_t` 递推地吸收历史信息。相较 full attention，状态大小不随序列长度线性增长，适合长上下文；它在 delta rule 上增加 channel-wise forget gate，让每个 key channel 有独立 retention。

值得注意的不是公式本身，而是 lower-bounded decay：

- chunkwise KDA 会用累计衰减的倒数缩放 key；
- 若 decay 可以无限接近 0，有限精度下容易 overflow；
- K3 将 log-decay 限制在下界内；
- 因此 causal diagonal tiles 也能转成 Tensor Core 上的 dense matrix multiplication。

这是典型的 algorithm-system co-design：数值参数化的改变直接消除 kernel 的慢路径。

### 为什么仍需要 MLA

线性 attention 擅长把过去压进固定状态，但固定状态天然存在容量和寻址限制。每 3 层插入一次 Gated MLA，相当于周期性恢复高容量全局交互。

因此 3:1 不是“谁替代谁”，而是：

- KDA 承担大多数长序列 mixing；
- MLA 提供选择性的全局 read；
- serving 时必须同时管理 KDA state 与 MLA KV cache。

## 2. Attention Residuals

普通 residual path 把前层结果顺序相加，远层信息必须经过多次变换才能到达深层。AttnRes 让当前模块用 learned pseudo-query 对 embedding 和前序 block outputs 产生权重，再选择性聚合。

直觉上，它把“深度方向的信息传递”从固定累加改成内容相关的 retrieval。

精读时要区分：

- layer mixing：访问哪些深度的表示；
- token mixing：KDA/MLA 在序列位置之间传递什么；
- channel mixing：MoE 在特征维度上变换什么。

K3 使用 block-level AttnRes 控制缓存量。其系统代价不是免费的：prefill 要避免每个 TP rank 重复 materialize block representations，decode 也需要 side-stream overlap 和 fused online-softmax merge。

## 3. Stable LatentMoE

K3 有 896 routed experts，每 token 激活 16 个，另有 2 个 shared experts。与 K2 相比，expert pool 增大 133%，active experts 增大 100%。

“Latent”来自先把 hidden state 从 7,168 投影到 3,584，再进入 routed experts。其目的包括：

- 在扩大 expert 数量时控制单 expert 计算；
- 让更大的 routed space 不必等比例放大每个 expert 的输入；
- 以更高 sparsity 扩大总参数。

但极端稀疏性会暴露三个稳定性问题。

### Normalized LatentMoE

对投影和路由相关表示做 normalization，抑制 latent projection 在 2.8T 规模下形成 ill-conditioned activation。

### SiTU-GLU

K3 用 Sigmoid Tanh Unit GLU 替代 SwiGLU。这里的要点不是记住激活函数名字，而是理解它作为极端规模下控制 gate/up branch 数值范围的一部分。

### Quantile Balancing

传统 auxiliary loss 只间接鼓励 expert balance，还可能污染主训练目标。Quantile Balancing 直接根据当前 batch 中 token 对各 expert 的 margin 分布，估计满足目标负载的 bias。

它的吸引力在于：

- 一次 forward pass 即可得到下一次 bias；
- 面向 pooled global batch，而非局部 shard；
- 不需要把 load-balancing loss 混入语言建模目标；
- 为 MoonEP 的静态 shape 和 perfectly balanced execution 提供更可控的上游路由。

## 4. Native Vision

MoonViT-V2 有 401M 参数、27 层、patch size 14。它不是在文本模型训练完后外挂，而是从预训练开始与语言 backbone 联合优化。

视觉侧的工程要点：

- spatial / temporal passes 处理图像和视频；
- temporal pooling 压缩视频 token；
- 2×2 pixel shuffle 将视觉 token 数减少 4 倍；
- 最高 3584×3584 的输入仍可进入 1M context。

这意味着 K3 的 multimodal agent 能力不是只依赖后训练工具调用，而是 base model 已学习统一的视觉-文本表示。

## 可复用结论

- Hybrid attention 的价值要与 cache 设计和 kernel 路径一起评估，不能只看理论复杂度。
- 深度扩展不一定只靠更深的 sequential residual；layer-wise retrieval 是另一条 scaling axis。
- 超大 MoE 的关键问题从“选几个 expert”转向数值稳定、全局负载估计和静态执行形状。
- K3 的 2.5x scaling efficiency 是组合收益，报告没有给出单模块归因。

## 未回答问题

- 3:1 的 KDA/MLA 比例是否最优？不同长度下最优比例是否不同？
- AttnRes 在相同参数/FLOPs 下的净增益和额外 memory traffic 各是多少？
- Quantile Balancing 对 expert specialization 是否有副作用？
- 896×16 的路由结构相比更少、更大的 experts，能力和 serving latency 如何权衡？
- NoPE + recurrent decay 对精确位置、逆序关系和多跳定位的误差曲线怎样？
