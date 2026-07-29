# 专题 2：预训练、Scaling Law 与百万上下文

## 核心判断

报告中的“2.5x scaling efficiency”和“1M context”是两个不同结论：

- 2.5x 是架构、数据与训练 recipe 相对 K2 的整体 scaling-law 拟合结果；
- 1M 是通过 NoPE、长数据构造和 progressive extension 获得的上下文能力。

前者不等于推理提速 2.5x，后者也不等于每个 1M token 任务都能稳定利用全程信息。

## 1. 数据构成

文本预训练包含四个主域：

- Web Text
- Code
- Mathematics
- Knowledge

每个域分别使用 rule-based filtering、质量分类器、去重和小模型 ablation 确定 sampling rate。知识和数学语料沿用 K2 的 rephrasing recipe：多风格、多视角改写，chunk-wise autoregressive generation，再对源文档做 fidelity verification。

视觉语料包括：

- caption；
- interleaved image-text documents；
- OCR 与 perception；
- video；
- visual coding。

visual coding 尤其值得关注：代码与渲染结果成对，覆盖 SVG、3D asset、网页、游戏和 CAD。这种数据直接对应 K3 后续“vision in the loop”的工程任务。

## 2. Scaling Law

K3 的架构变化改变了最佳训练区间，因此团队重新搜索：

- batch size；
- peak learning rate；
- tokens-per-parameter ratio；
- model shape。

报告在 held-out OOD validation loss 上拟合 K2/K3 scaling curve，声称 K3 组合方案以约 1/2.5 的 FLOPs 达到相同 loss。

这里要谨慎：

- 它是 K2 到 K3 整套 model family 的比较；
- 不只是 KDA 的收益；
- 没有公开拟合系数、置信区间、训练 token 总量和单项消融；
- validation loss 的效率不必然等比例转化为 agent benchmark 或 serving cost。

### Cosine vs. WSD

报告认为 cosine decay 优于 Warmup Stable Decay，但给出的重要方法论提醒比结论更有价值：不同 scheduler 的最佳 learning rate 和 batch size 不同，不能共用一组超参数做“公平”比较。K3 对两种 schedule 分别搜索后选择 cosine decay。

最终训练 recipe：

- Per-Head Muon；
- K2 weight clipping；
- Quantile Balancing；
- cosine decay；
- 1% linear warmup；
- weight decay 0.1。

## 3. Native Multimodal Pre-Training

K3 从训练开始共同优化语言和视觉，不采用“先训练 LLM、后挂视觉 encoder”的 post-hoc alignment。

视觉 token 和文本 token 在同一 next-token prediction objective 中交错。这个选择的假设是：若最终目标是 agent 在截图、图表、网页、CAD 和工具结果之间循环，视觉能力就不应只是一个后期 adapter。

报告没有回答：

- vision/text token 比例；
- 视觉 encoder 与 backbone 的 learning-rate ratio；
- native training 相对 staged alignment 的 controlled ablation；
- 视频与 visual coding 数据对最终 agent benchmark 的独立贡献。

## 4. 从 8K 到 1M

训练上下文分阶段扩展：

1. 预训练从 8K 开始；
2. 后续阶段扩到 64K；
3. cooldown 中从 256K 扩到 1M。

KDA 不使用显式 positional embedding，而用 recurrent gating 与 decay 隐式编码位置，因此无需 RoPE rescaling 或 interpolation 就能外推到 1M。

### 长数据为什么不能只拼接

天然长文档和视频包含大量低质量内容：

- near-duplicates；
- binary blobs；
- truncated files；
- invalid generated logs；
- 冗余视频片段。

K3 使用 exact/fuzzy deduplication、frame perceptual hashing、启发式/分类器过滤和结构校验。真正长且连贯的数据稀缺，因此还要 upsample。

更关键的是，长度不自动产生长程依赖。K3 额外构造 synthetic long-context data：

- 重排并拼接 multimodal documents 与 subtasks；
- 让任务必须访问散布于整个 context 的信息；
- 避免模型只在长输入的局部窗口内工作。

## 5. 需要怎样验证“1M”

精读或复现时至少分四层测试：

| 层次 | 要验证什么 |
|---|---|
| 接受长度 | API/engine 能否真的处理 1,048,576 tokens |
| 定位 | 远距离 needle、精确引用、顺序与位置敏感任务 |
| 推理 | 跨段聚合、多跳、冲突消解、跨模态关联 |
| Agent | 多轮工具调用、状态持续、错误恢复和 context management |

报告最强的论点在第四层：它确实把百万 token 当作 long-horizon rollout 的累计工作上下文。但 BrowseComp 的 91.2 使用 300K 触发的 context compaction；完整 1M 且无 context management 时为 90.4。这个细节说明“更长原始窗口”和“更好的上下文管理”仍是两个能力。

## 可复用结论

- 长上下文训练要同时解决位置机制、数据质量、依赖跨度和 serving cache。
- scheduler 对比必须分别调参，否则结论可能只是 hyperparameter alignment。
- native multimodal 的价值应由 agentic visual tasks 验证，而不只看静态 VQA。
- scaling law 的整体收益不能替代 component-level ablation。

## 未回答问题

- 总训练 token、各域比例、vision token 比例和 cooldown token 数是多少？
- 1M 数据中自然长文档与合成长任务的比例是多少？
- KDA NoPE 在超过训练长度时还能否稳定外推？
- 2.5x loss efficiency 对 downstream、wall-clock 和能耗分别转化成多少？
- 300K context compaction 的具体策略及其独立贡献是什么？
