# 专题 3：Agentic Post-Training 与多档推理强度

## 总览

K3 的 post-training 可以压缩成：

```text
SFT cold start
  -> 3 domains × 3 reasoning efforts = 9 RL experts
  -> Multi-Teacher On-Policy Distillation
  -> one unified deployment model
```

这条路线把“能力专业化”和“最终部署统一”分开处理。

## 1. SFT：先得到能行动的 cold start

SFT 数据由前代 Kimi 的 domain-specialized models 合成，再经过 multi-stage verification 和 human-in-the-loop annotation。数据统一序列化为 XTML chat template，以表示复杂 tool call 和 agent trajectory。

SFT 的目标不是直接达到最终 benchmark，而是初始化：

- adaptive reasoning；
- precise tool calling；
- long-horizon execution；
- 后续 RL 所需的基本探索能力。

从 SFT 开始就进行量化感知训练：MoE expert weights 使用 MXFP4，输入 activation 使用 MXFP8，非 expert 模块保持更高精度。这避免最后才量化导致 agent 行为显著漂移。

## 2. RL：领域与 effort 解耦

三个领域：

1. General tasks：general experience、vision、reasoning、faithfulness、search、knowledge work。
2. General agents：long-horizon assistant、deep research、paragraph-level writing。
3. Coding agents：SWE、coding experience、kernel、web development。

每个领域分别训练 low/high/max 三档 effort，共 9 个专家。

这个设计隐含两个判断：

- 同一个 reward/data mix 很难同时优化所有能力；
- reasoning effort 不是纯 inference parameter，而是需要在 policy training 中显式塑造。

Figure 8 显示，随着 RL FLOPs 增加，多个能力分数与平均 assistant steps 一同上升。它支持“更强 agent 学会执行更长轨迹”，但不能单独证明“步数越多导致能力越强”，两者可能共同由更好的 policy 引起。

## 3. Partial Rollout：不等最慢轨迹

每轮对 N 个 prompts 各采样 K 条 completion，共有 `N × K` 条 active trajectories。当完成比例达到 λ 时暂停 generation，先优化已完成组；未结束轨迹入队，下一 iteration 优先恢复。

收益：

- 避免 long-horizon straggler 卡住同步训练；
- 保留未完成轨迹，不浪费已经执行的工具调用；
- 让超长 trajectory 横跨多个 policy iteration。

代价是 data staleness：轨迹后半段可能由旧 policy 生成。K3 使用 per-token regularization 把 policy update 限制在局部邻域，以容忍 extreme off-policy data。

这与 SAO 的 single-rollout asynchronous optimization 是不同路线：

- K3 仍以同一 prompt 的 K 条完成为优化单元；
- SAO 试图去掉 group barrier，并引入 critic/importance sampling；
- 两者共同说明长程 agent RL 的核心瓶颈之一是 rollout 到达协议。

## 4. Reasoning Effort RL

对每个问题 `x`，cold-start model 估计初始预算 `b0(x)`。若 trajectory 的消耗 `T(y)` 超过 `τ × b0(x)`，reward 被覆盖为 -1。

- general task 的 `T(y)` 是 thinking tokens；
- agentic task 的 `T(y)` 是累计 output tokens，包括 reasoning 和 tool arguments。

先用较大的 τ 训练 max expert，但仍设置上限防止 overthinking；再逐步降低 τ，得到 high 和 low experts。不同 domain 的 τ 由 human-in-the-loop 调整。

这比推理时硬截断更合理，因为模型在训练中学习“预算内完成任务”。但报告未给出：

- 每档 effort 的预算分布；
- 相同成功率下节省多少 token；
- low/high/max 的 Pareto 曲线；
- budget penalty 是否鼓励跳过必要验证。

## 5. Agentic Generative Reward Model

对不可程序验证的 general tasks，K3 使用 tournament-style binary comparison。judge 必须：

1. 阅读 outcome/product/text；
2. 生成 rubric；
3. 按 rubric 给 candidate 评分；
4. 把分数记录到 scorepad。

为防止 reward hacking 到冗长输出，若输出长度超过 cold-start verbosity 的 `σ` 倍，candidate 自动输掉比较。

这个做法的亮点是让 rubric generation 成为 reward protocol 的显式步骤；风险则是 judge 自己的 rubric 偏差、长度偏好和 domain blind spot 会进入 reward。

## 6. MOPD：把 9 个专家合并回来

对 sampled domain `d` 和 effort `e`，用对应 teacher 的 token probability 与 student probability 的 log ratio 形成 clipped dense reward：

```text
r_opd = clip(stop_gradient(log π_teacher / π_student), -Rmax, Rmax)
```

它是 on-policy 的：student 生成自己当前分布下的 trajectory，teacher 对这些 token 提供 dense guidance。相较离线蒸馏，student 较少遇到纯 teacher distribution 与部署 distribution 的错位。

报告称更细粒度的 top-k distillation objective 没有带来明显收益，最终采用上述 token reward。

## 可复用结论

- Domain specialization 与 deployment consolidation 可以分阶段优化。
- Effort level 应被视为 policy dimension，而不只是 max_tokens 参数。
- Long-horizon RL 算法必须与 rollout persistence、sandbox resume 和 off-policy tolerance 一起设计。
- Generative reward 的 rubric、scorepad 和 verbosity budget 都是可审计的 reward protocol 元件。

## 未回答问题

- 9 个 experts 相比一个 joint RL model 的收益是多少？
- MOPD 是否会平均掉某个 expert 的峰值能力？
- λ、K、staleness 与 policy regularization 的消融在哪里？
- effort conditioning 是输入控制、隐式 policy mode，还是两者结合？
- judge rubric 是否保存并用于 reward audit？
- QAT 对长轨迹中的 tool-call syntax 和 rare expert behavior 有何影响？
