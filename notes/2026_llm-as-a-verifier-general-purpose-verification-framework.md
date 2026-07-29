# LLM-as-a-Verifier: A General-Purpose Verification Framework

- Year: 2026
- Date: 2026-07-07
- Venue: arXiv / alphaXiv; NeurIPS 2026 submission
- Authors: Jacky Kwok, Shulu Li, Pranav Atreya, Yuejiang Liu, Yixing Jiang, Chelsea Finn, Marco Pavone, Ion Stoica, Azalia Mirhoseini
- Links:
  - alphaXiv: https://www.alphaxiv.org/abs/2607.05391
  - arXiv: https://arxiv.org/abs/2607.05391
  - PDF: https://arxiv.org/pdf/2607.05391
  - Project: https://llm-as-a-verifier.com/
  - Code: https://github.com/llm-as-a-verifier/llm-as-a-verifier
  - TurboAgent: https://github.com/llm-as-a-verifier/TurboAgent
- Local PDF: [../papers/2026_llm-as-a-verifier-general-purpose-verification-framework.pdf](../papers/2026_llm-as-a-verifier-general-purpose-verification-framework.pdf)
- Tags: agent, verification, trajectory-reward-model, test-time-scaling, llm-as-a-judge, best-of-n, progress-estimation, dense-reward
- Status: reading
- Rating:
- One-line takeaway: LLM-as-a-Verifier 不再把评分分布压成一个离散 token，而是对评分 token 的概率分布取期望，再沿评分粒度、重复评估和标准分解三条轴扩大 verification compute，并用低于全量两两比较成本的 Pivot Tournament 选择最佳 agent trajectory。

## Problem

Best-of-N 和 test-time scaling 的上限常常不是“模型能否生成正确解”，而是“系统能否从多条候选 trajectory 中认出正确解”。论文在 Terminal-Bench V2 上汇总大量候选时观察到 Oracle Pass@K 可到 98.9%，说明生成侧已经包含大量潜在正确答案，但选择器无法完全利用这部分 headroom。

标准 LLM-as-a-Judge 通常让模型输出 1–5 或 1–10 的单个分数，再取概率最高的 score token 作为最终结果。复杂 agent trajectory 很容易被压到同一个整数，Terminal-Bench V2 的单次离散 judge 比较中 tie rate 达到 26.7%。训练专用 reward model 可以提供连续信号，但又受训练数据和领域迁移限制。

论文因此研究三个问题：

1. 不训练新 reward model，能否从通用 LLM 中提取更细的 verification signal？
2. verification compute 增加时，哪些维度会稳定提升判断准确率？
3. 当候选数 N 增大时，怎样避免全量 pairwise comparison 的 O(N²) 成本？

## Core Idea

LLM-as-a-Verifier 的核心是保留模型在 score token 上表达的不确定性。给定任务、评价标准和两条候选 trajectory，模型仍然输出有限评分等级，但系统不取 argmax token，而是对所有评分 token 的概率加权求期望，得到连续 reward。

这使 verification 可以沿三条相互补充的轴扩展：

- Score granularity `G`: 增大可用评分等级，让相近的内部判断不必被舍入到同一个 token。
- Repeated evaluation `K`: 多次独立评估取平均，降低单次采样和 prompt 偏差带来的方差。
- Criteria decomposition `C`: 把复合的“是否正确”拆成 specification、output、errors 等更简单标准，再对结果做 ensemble。

为了从 N 条候选中选出一条，论文把连续 reward 转成 Bradley–Terry pairwise preference，并提出 Probabilistic Pivot Tournament：先用一个随机环做低成本初筛，再只让所有候选与少量 top-k pivots 比较，把复杂度从 O(N²) 降到 O(Nk)。

## Method

### 1. Fine-Grained Reward Estimation

设评分 token 集合为：

```text
V_score = {v_1, ..., v_G}
```

对任务 `x`、trajectory `tau`、标准 `c`，verifier 在评分位置给出 token 分布。连续 reward 为：

```text
R(x, tau) = 1 / (C K)
            * sum_c sum_k sum_g
              p_theta(v_g | x, c, tau) * phi(v_g)
```

其中：

- `G` 是评分粒度；
- `K` 是重复验证次数；
- `C` 是评价标准数；
- `phi(v_g)` 把 score token 映射为标量。

论文先把 `R` 线性归一化到 `[0, 1]`，再用 Bradley–Terry 形式把两条 trajectory 的 reward 差转成软偏好：

```text
P(tau_i > tau_j | x) = sigmoid(R(x, tau_i) - R(x, tau_j))
```

这种做法和离散 judge 的关键差异不是“输出更多小数位”，而是不丢弃 score token 分布里的次优概率质量。一个 5 分最可能、4 分也有很高概率的判断，会和几乎确定为 5 分的判断得到不同连续值。

### 2. Verification Scaling

三条 scaling 轴分别针对不同误差来源：

- `G` 增大 score separation。Terminal-Bench V2 pairwise accuracy 从 `G=1` 的 73.1% 提高到 `G=20` 的 77.5%，SNR 从 0.775 提高到 0.799。
- `K` 对多个 reward estimator 求平均，理论上方差按 O(1/K) 下降；实验从 `K=1` 的 74.7% 提高到 `K=16` 的 77.5%，但后期收益递减。
- `C` 把代码 agent 的正确性拆成 Specification、Output、Errors。单项准确率为 75.2%–76.4%，三项 ensemble 达到 78.3%。

`query-optimize` case study 很直观：离散 1–5 judge 在 100 次比较中出现 88 次 tie；对同一个 5 档分布取期望后没有 tie，并在 69 次中排对；扩大到 20 档后排对 77 次。

### 3. Probabilistic Pivot Tournament

PPT 的流程是：

1. 把 N 条候选随机排成 Hamiltonian cycle，相邻候选比较一次。
2. 每条候选恰好在 prompt 的 A、B 位置各出现一次，以抵消 position bias。
3. 按 ring pass 的平均 win mass 排序，取 top-k 作为 pivots。
4. 比较所有 non-pivot–pivot pairs，并补齐 pivot–pivot pairs。
5. 聚合 soft win mass `w_i`，除以参与比较次数 `c_i`，返回 `argmax w_i / c_i`。

比较次数约为：

```text
N + k(N-k) + k(k-1)/2 = O(Nk),  k << N
```

在 89 个 Terminal-Bench V2 任务、每题 20 条候选的设置中，`k=5` 用 6,609 次 pair query 达到 66.27%，full round-robin 用 13,111 次达到 67.42%；`k=9` 用 9,630 次达到 67.13%。

### 4. Progress Signal 和 Dense Reward

同一个 verifier 还被用来给 trajectory prefix 打分。论文用 Value-Order Correlation 衡量 prefix score 与步骤时间顺序的 Spearman 相关性：成功的 Terminal-Bench trajectory 为 0.848，失败 trajectory 为 0.769；机器人轨迹上达到 0.966。

在 RL 中，论文把该连续信号当作 reward shaping：

- LIBERO + DSRL-SAC: 用 vision-language verifier 给 trajectory frames 生成 progress reward，再与环境 reward 相加。
- MATH + GRPO: 用 PPT 评价 reasoning trace，把 reasoning reward 加到 correctness 和 format reward。

## Experiments

### Test-Time Selection

默认配置使用 `G=20`、`K=8` 和三项 criteria decomposition。文本 agent 实验主要以 Gemini 2.5 Flash 为 verifier；机器人视频使用 Qwen 3.6 35B VLM。

| Benchmark | Candidate-pool Pass@1 | Oracle Pass@N | LLM-as-a-Verifier |
|---|---:|---:|---:|
| Terminal-Bench V2 | 83.1% | 92.1% | **86.5%** |
| SWE-Bench Verified | 76.1% | 84.4% | **78.2%** |
| MedAgentBench | 70.2% | 75.0% | **73.3%** |

具体候选池：

- Terminal-Bench V2: Capy scaffold，GPT-5.5 每题 5 条 trajectory。
- SWE-Bench Verified: mini-swe-agent，每题各取 Claude Opus 4.5、Gemini 3 Flash、MiniMax M2.5 一条 trajectory。
- MedAgentBench: AgentBench scaffold，Claude Opus 4.8 每题 5 条 trajectory。

在 RoboRewardBench 的 500 对机器人视频上，zero-shot LLM-as-a-Verifier 达到 87.4% preference accuracy，高于离散 LLM-as-a-Judge 的 70.8%、RoboReward-8B 的 81.4%、Robometer-4B 的 78.8% 和 TOPReward 的 74.7%。

### RL Sample Efficiency

- LIBERO ketchup task: verifier-shaped DSRL-SAC 达到相同成功率所需环境步数约减少到 sparse baseline 的 1/1.8，最终 success rate 为 0.76 vs. 0.69。
- MATH + Qwen3-8B + GRPO: reasoning reward 带来约 1.1× sample-efficiency improvement。

### Logprob 不可用时

方法原则上要求 score-token logprobs。对不暴露 logprobs 的 closed frontier model，论文提出两阶段 workaround：先让 closed model 生成领域判断 reasoning，再交给可读 logprob 的 Gemini 2.5 Flash 输出连续 reward。Terminal-Bench V2 上，单次评估从直接使用 GPT-5.5 离散分数的 74.9% 提高到 80.1%，tie rate 从 10.9% 降到 0。

## Strengths

- 抓住了离散 judge 的真实信息损失：同一个 forward pass 已经产生一个概率分布，取期望几乎不需要额外训练，却比 argmax score token 更细。
- 把 verification scaling 拆成 granularity、repetition、criteria 三个可独立控制的旋钮，并分别给出消融和误差解释。
- PPT 同时处理 position bias 和候选排名成本，比“任意选几个 anchor”更有结构。
- 验证范围跨代码、医疗、机器人视频，且同一个连续信号还能用于 selection、progress tracking 和 RL reward shaping。
- 公开了代码、数据、Python package 和 agent extension，复现实用性较好。

## Weaknesses

- 方法高度依赖 score-token logprobs；两阶段 workaround 需要再调用一个开放 logprob verifier，增加了系统复杂度、延迟和新的模型依赖。
- `G`、`K`、`C` 和 candidate count 都会增加 inference cost。PPT 只降低 candidate-ranking 的 pair 数，不能消除每个 pair 内多标准、多重复验证的乘法成本；论文没有给统一的 latency / token / dollar cost 曲线。
- “general-purpose”指框架不需要 per-domain training，不代表完全无领域设计：代码任务使用专门 criteria，视觉任务换成 VLM，医疗任务仍需要领域判断能力。
- 与 candidate-pool Pass@1 的提升在 Terminal-Bench 和 SWE-Bench 上分别只有 +3.4 和 +2.1 个百分点，明显没有吃满 +9.0 和 +8.3 的 oracle headroom；tie 消失不等于 verifier error 消失。
- leaderboard SOTA 对比混合了不同模型和 harness。最可信的因果对照是同一 candidate pool 上的 Pass@1 / Oracle / Ours，而不是跨 scaffold 的绝对排名。
- VOC 衡量“分数是否随时间上升”，不直接衡量剩余工作、最终成功概率或是否值得回滚；失败 trajectory 的 VOC 仍有 0.769，成功与失败的差距只有 0.079，不能直接把它当安全停止信号。
- RL 证据集中在一个 LIBERO ketchup task 和 MATH，sample-efficiency 结论能否扩展到更多机器人任务、agent 环境和 reward-hacking 场景仍不明确。

## Useful For

- 给 Best-of-N、parallel rollout 或 search-based agent 设计 trajectory selector。
- 改进现有 LLM-as-a-Judge：优先保留 score distribution，再考虑增加重复采样和 rubric ensemble。
- 为长程 coding agent 设计 progress monitor，但需要与 executable tests、environment state 和 completion checks 组合。
- 给 sparse-reward agent RL 构造 dense proxy reward，并分析 verifier bias 是否会进入 policy optimization。
- 和 SAO、OpenThoughts-Agent 放在一起看：它们都依赖 verifier reward，但分别处理 reward 信号、异步优化协议和训练数据 recipe。

## Questions

- 在固定 token、latency 和 dollar budget 下，增加 `G`、`K`、`C`、candidate count 和 pivot count，哪一项的边际收益最高？
- 连续期望消除 tie 后，是否会把模型本来不确定的判断伪装成精确小数？怎样做校准或 abstention？
- 如果 scoring tokens 的 tokenization、先验概率或顺序改变，reward calibration 是否稳定？
- PPT 的 ring pass 只能在期望上抵消 A/B position bias；单次任务里是否需要双向复评关键 pair？
- 代码任务有 executable verifier 时，LLM verifier 应该负责早期筛选、失败分类，还是只填补测试覆盖不到的语义标准？
- progress reward 被用于 RL 后，policy 是否会学会制造“看起来在推进”的轨迹，而不是真正完成任务？

## Share Notes

适合的分享主线：

1. Generation 已经给出大量正确候选，为什么 Best-of-N 仍吃不满 Oracle Pass@N？
2. Judge 的信息损失：argmax score token 如何制造 26.7% tie。
3. Verifier 的关键公式：对评分分布取期望，再沿 `G × K × C` 扩大 verification compute。
4. PPT：用 ring pass 校正位置偏差，用 pivots 把 O(N²) 降到 O(Nk)。
5. 三种用途：test-time selection、progress tracking、dense RL reward。
6. 工程边界：logprob、延迟、领域 criteria、跨 harness 对比和 reward hacking。
