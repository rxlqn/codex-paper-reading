# Kimi K3: Open Frontier Intelligence

- Year: 2026
- Date: 2026-07-28
- Venue: Technical Report
- Authors: Kimi Team
- Links:
  - Tech blog: https://www.kimi.com/blog/kimi-k3
  - Report: https://github.com/MoonshotAI/Kimi-K3/blob/main/k3_tech_report.pdf
  - Model: https://huggingface.co/moonshotai/Kimi-K3
  - Code: https://github.com/MoonshotAI/Kimi-K3
- Local PDF: [../papers/2026_kimi-k3-open-frontier-intelligence.pdf](../papers/2026_kimi-k3-open-frontier-intelligence.pdf)
- Tags: open-weight, frontier-model, MoE, linear-attention, long-context, multimodal, agentic-RL, systems
- Status: reading
- Rating:
- One-line takeaway: Kimi K3 不是只靠把 MoE 扩到 2.8T 参数，而是同时扩大预训练基础、测试时 reasoning/agent horizon 与承载百万 token 轨迹的系统，把架构、RL 环境和基础设施共同变成能力的一部分。

## 精读导航

这份 47 页报告包含至少六条可以独立追踪的研究线，已拆成专题：

1. [总阅读路线](../topics/kimi-k3/README.md)
2. [架构：KDA、AttnRes 与 Stable LatentMoE](../topics/kimi-k3/01-architecture.md)
3. [预训练、Scaling Law 与百万上下文](../topics/kimi-k3/02-pretraining-and-long-context.md)
4. [Agentic Post-Training 与多档推理强度](../topics/kimi-k3/03-agentic-post-training.md)
5. [RL 任务合成与长程 Agent 环境](../topics/kimi-k3/04-rl-environments.md)
6. [3T 训练、1M RL 与在线推理基础设施](../topics/kimi-k3/05-infrastructure-and-serving.md)
7. [评测、SOTA 判断与复现边界](../topics/kimi-k3/06-evaluation-and-sota.md)

## Problem

开放权重模型在 test-time scaling、reasoning RL 和 agent training 上进步很快，但预训练基础大多仍停留在 1T 参数级附近。若大家继续在相似规模的 base model 上叠加后训练，开放模型的能力可能收敛，而与最强闭源模型的差距重新扩大。

K3 的问题因此不是单纯“如何训练更大的 MoE”，而是：

- 如何同时扩大 pre-training compute 与 test-time compute；
- 如何让 1M 上下文不只存在于规格表，而能承载数百到数千次工具调用；
- 如何把不同领域、不同 reasoning effort 的 RL 专家合并回一个统一模型；
- 如何让 2.8T MoE 的训练、百万 token RL 和在线服务在工程上可执行。

## Core Idea

K3 把信息流沿三个维度扩展：

- 序列维度：每个 block 用 3 层 KDA + 1 层 Gated MLA，在长序列效率和全局交互之间折中；
- 深度维度：AttnRes 允许模块对 embedding 和之前 block 的表示做选择性读取；
- 宽度维度：Stable LatentMoE 在 896 个 routed experts 中每 token 激活 16 个，并解决极稀疏路由的稳定性和负载均衡。

随后，K3 用 SFT 建立 agent cold start，在 general、general agent、coding 三个领域和 low/high/max 三档 reasoning effort 上训练 9 个 RL expert，再通过 Multi-Teacher On-Policy Distillation 合并成统一模型。整套训练由 3T MoE、百万 token partial rollout、持久化 sandbox 与混合 KDA-MLA serving 系统支撑。

## Model Card

| 维度 | Kimi K3 |
|---|---|
| Architecture | MoE，native multimodal |
| Total / activated parameters | 2.78T / 104.2B |
| Layers | 93 |
| Attention | 69 KDA + 24 Gated MLA |
| Hidden dimension / heads | 7,168 / 96 |
| Routed / active / shared experts | 896 / 16 / 2 |
| MoE expert hidden dimension | 3,072 |
| Vision encoder | MoonViT-V2，401M |
| Context | 1,048,576 tokens |
| Position encoding | NoPE |
| Deployment precision | MXFP4 expert weights + MXFP8 activations，post-training QAT |

## Method

### Architecture

- Hybrid Attention:
  - KDA 用 recurrent state 和 channel-wise decay 做线性复杂度的长序列混合；
  - 每 3 层 KDA 插入 1 层 Gated MLA，末层额外保证一次全局 attention；
  - lower-bounded decay 让 chunkwise KDA 的 causal tiles 可以使用 Tensor Core dense GEMM。
- AttnRes:
  - 不只累计前一层 residual，而是对 embedding 和前序 blocks 的表示做 learned attention；
  - block-level 设计控制存储和读取成本。
- Stable LatentMoE:
  - 先投影到 0.5x latent dimension，再在 896 experts 中路由；
  - normalization、SiTU-GLU 和 Quantile Balancing 分别处理 activation、GLU 与 expert load 的稳定性问题。
- Native vision:
  - MoonViT-V2 从训练开始就与语言 backbone 联合优化；
  - pixel shuffle 将视觉 token 数减少 4 倍，使高分辨率图像可进入长上下文。

### Pre-Training

- 数据覆盖 Web Text、Code、Mathematics、Knowledge 和大规模 vision corpus。
- 文本使用规则、质量分类器和去重；知识/数学数据进行多风格 rephrase 并验证忠实度。
- 视觉数据包括 caption、交错图文、OCR、perception、video 和 visual coding；后者覆盖 SVG、3D、网页、游戏、CAD。
- 训练使用 Per-Head Muon、weight clipping、Quantile Balancing、cosine decay、1% warmup 和 0.1 weight decay。
- 上下文从 8K 逐步扩展至 64K，并在 cooldown 从 256K 扩到 1M；KDA 的 NoPE 设计避免 RoPE rescaling/interpolation。
- 架构、数据与训练 recipe 合计带来报告所称的约 2.5x scaling efficiency improvement；该数字是相对 K2 的拟合 scaling-law 结果，不应直接理解成推理速度提升 2.5x。

### Post-Training

- SFT:
  - 用前代 Kimi 的领域模型合成复杂 agent trajectory；
  - multi-stage verification + human-in-the-loop annotation；
  - 统一序列化为 XTML chat template；
  - 从 SFT 开始就做 MXFP4/MXFP8 QAT。
- RL:
  - 三领域：general tasks、general agents、coding agents；
  - 三档 effort：low、high、max；
  - 总计 9 个 domain-effort experts。
- Partial rollout:
  - 不等待所有长轨迹结束；完成比例达到阈值后即开始优化；
  - 未完成轨迹排队并在后续 iteration 恢复；
  - per-token regularization 用于承受跨 iteration 的 stale/off-policy data。
- Reasoning effort:
  - 以 cold-start model 的每题初始预算为基准；
  - 超出倍数阈值的 trajectory reward 直接设为 -1；
  - 逐步 anneal budget multiplier 得到 max/high/low experts。
- MOPD:
  - 从对应 domain-effort teacher 的 token probability 得到 clipped dense OPD reward；
  - 将 9 个专家合并为单一 student。

### RL Environment

- 可组合 white-box harness：tool、system prompt、context management、skills、memory、subagents 都是可配置模块，训练时动态实例化 Kimi Code、Claude Code、Codex、OpenClaw、Hermes 等协议。
- 自演化 knowledge graph：从粗粒度概念递归探索到 atomic concepts，再基于节点组合检索真实材料并合成任务。
- 任务覆盖搜索、专业知识工作、软件工程、kernel optimization、视觉推理、长期 personal assistant、autonomous execution 和 web development。
- 重点不是“生成长轨迹”本身，而是让长轨迹有 sandbox 状态、可验证 outcome 和多种 harness 分布。

## Infrastructure

- Pre-training:
  - PP/VP、EP、TP、SP 与 ZeRO-style data parallelism 组合；
  - MoonEP 通过 bounded redundant experts、静态计算形状和 zero-copy communication 追求 perfectly balanced expert execution；
  - activation offloading、memory-efficient optimizer 和 multimodal encoder 优化控制显存。
- 1M agentic RL:
  - co-located rollout/training，实验控制在数百 GPU；
  - external KV cache pool 保留超长 partial rollout；
  - adaptive throttling 在 generation 和 training 间动态分配资源；
  - microVM sandbox 保存并恢复文件系统、进程和环境状态。
- Serving:
  - KDA recurrent state 与 MLA KV cache 统一 paged 管理；
  - 物理块和 prefix hash 粒度解耦，在 512-token 边界复用长 prefix；
  - 专用 KDA、Block AttnRes、Stable LatentMoE kernels；
  - cache-aware affinity scheduling + budget-based admission control。

## Experiments

### 代表性结果

| 能力 | K3 结果 | 解读 |
|---|---:|---|
| ProgramBench | 77.8 | 表中第一 |
| Terminal-Bench 2.1 | 88.3 | 接近 GPT-5.6 Sol 88.8 |
| FrontierSWE | 81.2 | 第二，低于 Claude Fable 5 的 86.6 |
| SWE-Marathon | 42.0 | 表中第一 |
| BrowseComp | 91.2 | 表中第一；完整 1M 且无 compaction 时为 90.4 |
| MCPMark-Verified | 94.5 | 表中第一 |
| AutomationBench | 30.8 | 表中第一 |
| SpreadsheetBench 2 | 34.8 | 表中第一 |
| Harvey Lab-AA | 94.6 | 表中第一 |
| GPQA Diamond | 93.5 | 与 GPT-5.5 并列，低于 GPT-5.6 Sol |
| OmniDocBench | 91.1 | 表中第一 |
| WebDev Arena | 1,678 Elo | 报告截点时第三方榜单第 1 |
| Artificial Analysis Intelligence Index v4.1 | 57.1 | 报告截点时第 4/580 |

### “当前 SOTA 开源模型”怎么表述更准确

截至 2026-07-28，可以把 K3 称为“当前开放权重模型的 frontier / 综合 SOTA 候选”，尤其在 agentic、coding、long-horizon 和 multimodal tool-use 的组合能力上。报告中的开放对手 GLM-5.2 在所列大多数项目上落后，第三方 Vals Index、Artificial Analysis 与 WebDev Arena 也支持它处于第一梯队。

但不宜写成“所有能力都绝对第一”：

- K3 自己明确承认总体仍落后于最强闭源模型 Claude Fable 5 和 GPT-5.6 Sol；
- 即使在开放模型内部，“SOTA”也取决于 benchmark、effort、harness、工具和成本；
- 主表混合了官方自测、in-house benchmark、第三方分数与不同 agent harness；
- 多数 K3 结果使用 `max` effort，不能与默认档位或同 token budget 直接等价；
- “open-weight”比“完全开源”更精确：权重已发布且许可较宽松，但报告没有公开完整训练数据、训练代码和全部 recipe。

## Strengths

- 把架构、训练算法、agent environment 和系统共同写进同一条因果链，而不是只给模型卡和 leaderboard。
- 3T MoE、1M context 与 long-horizon RL 都提供了足够多的系统细节，特别是 partial rollout、external KV、sandbox resume 和 KDA cache。
- RL 的 domain × effort 专家矩阵再用 MOPD 合并，是一种清晰的能力组合范式。
- 不只关注静态 QA，重点评估 coding、knowledge work、office deliverables、computer use 和专业 workflow。
- 报告主动披露 harness、fallback、cyberguard、context compaction 和第三方截点等评测条件。

## Weaknesses

- 没有披露预训练 token 数、完整数据配比、训练总 FLOPs、RL 样本规模或主要 ablation 数字，难以严格复现。
- 2.5x scaling efficiency 是多个改动合并后的整体结论，无法归因到 KDA、AttnRes、Stable LatentMoE、数据或 optimizer 中的单一因素。
- Agent 评测高度依赖 harness；主表中 Kimi Code、Claude Code、Codex 和 Terminus 2 并存。
- 多个 benchmark 为 in-house，第三方表也按各来源原始设置汇总，不是统一协议下的完全可比实验。
- 报告缺少系统性的 safety、failure mode、long-context degradation 与真实部署可靠性分析。

## Useful For

- 研究“长程 Agent 能力究竟来自模型、轨迹、环境还是系统”的统一案例。
- 对比 OpenThoughts-Agent 的 data recipe、Agents-A1 的 horizon scaling、SAO 的异步 RL 和 LLM-as-a-Verifier 的 reward design。
- 设计跨 harness 的 agent RL 环境，以及 domain/effort experts 的 consolidation。
- 设计 hybrid linear/full attention 在百万上下文下的训练与 serving。
- 做一次 30-45 分钟分享：重点不是参数规模，而是“为什么 frontier agent model 必须是 algorithm-system co-design”。

## Questions

- 2.5x scaling efficiency 中，架构、数据和训练 recipe 各贡献多少？
- KDA 的 NoPE 外推到 1M，在信息定位、顺序敏感和长程干扰上分别有什么退化曲线？
- MOPD 相比直接混合 9 个 experts 的 trajectories 做 SFT/RL，增益在哪里？
- Partial rollout 的 λ、policy staleness 和 per-token regularization 强度如何共同影响收敛？
- Knowledge graph synthesis 是否会把覆盖度做大，却把真实任务分布做得更“教材化”？
- 同一模型换成统一 harness 后，主表排名会改变多少？
- 104B activated parameters 与 MXFP4 权重对自部署成本意味着什么？“开放权重”是否真的等于可普遍部署？
- 报告没有单独列 limitations；哪些安全与可靠性结论尚不能从 benchmark 推出？

## Share Notes

推荐标题：**Kimi K3 真正扩大的不是 2.8T 参数，而是 Agent 的可训练时间尺度**

分享主线：

1. 先用 2.8T / 104B / 1M 建立规模感；
2. 再解释 token、layer、channel 三维信息流；
3. 用 3 domains × 3 efforts = 9 experts 讲 post-training；
4. 用 partial rollout + KV retention + resumable sandbox 说明“长程 RL 是系统问题”；
5. 最后用评测条件审计 SOTA，避免只读排行榜。
