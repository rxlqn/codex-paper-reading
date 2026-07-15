# Agent RL 为什么不该等齐 8 条 Rollout？

- Paper: Single-Rollout Asynchronous Optimization for Agentic Reinforcement Learning
- Note: [../../../notes/2026_single-rollout-asynchronous-optimization-for-agentic-reinforcement-learning.md](../../../notes/2026_single-rollout-asynchronous-optimization-for-agentic-reinforcement-learning.md)
- PDF: [../../../papers/2026_single-rollout-asynchronous-optimization-for-agentic-reinforcement-learning.pdf](../../../papers/2026_single-rollout-asynchronous-optimization-for-agentic-reinforcement-learning.pdf)
- Target audience: AI 硕博 / 大厂 AI 工程师
- Draft status: draft
- Suggested format: 8-9 张图 + 正文
- Keywords: agentic RL, asynchronous RL, SAO, GRPO, single rollout, value model, off-policy

## 0. 一句话定位

这篇论文的核心不是简单把 GRPO 的 group size 改成 1，而是重新设计异步 agent RL 的训练协议：一条 rollout 完成就更新，并用严格的 token trust region、更强的 critic 和 skip-observation GAE 把 single-rollout 的高方差控制住。

## 1. 标题候选

- Agent RL 为什么不该等齐 8 条 Rollout？
- GRPO 不适合异步 Agent 训练？SAO 的 Single-Rollout 解法
- 一条轨迹就更新：长程 Agent RL 如何摆脱 Group Barrier

## 2. 开头 Hook

长程 agent 的 rollout 长度差异非常大。

有的 coding task 很快结束，有的要调用工具、读环境反馈、修改代码几十甚至几百轮。同步训练必须等 batch 里最慢的轨迹完成，前面已经结束的 GPU 和样本都在等待。

异步 RL 看起来能解决这个问题：一条 trajectory 完成，就立刻送去训练。

但如果算法是 GRPO，矛盾又回来了。GRPO 需要同一个 prompt 的多条 rollout 凑成一组，再用组内 reward 做相对 advantage。最慢的那条还是会拖住整组，而且等待时间越长，先完成的 rollout 离当前 policy 越远。

SAO 这篇论文问的是：能不能每个 prompt 只采一条 rollout，完成即训练，同时还能稳定跑一千步？

## 3. 问题定义

### 3.1 两个时钟不一致

- Rollout engine 持续异步生成 trajectory。
- Trainer 持续更新当前 policy。
- 一条长 trajectory 生成期间，trainer 可能已经更新多个版本。

因此训练时看到的是：

```text
trajectory ~ pi_rollout
update      -> pi_theta

pi_rollout != pi_theta
```

这就是 policy lag。轨迹越慢、等待越久，off-policy drift 越严重。

### 3.2 GRPO 的 group barrier

```text
one prompt
  -> rollout 1  -- done
  -> rollout 2  -- done
  -> ...
  -> rollout 8  -------- done

wait until all 8 are ready
then compute group-relative advantage
```

对真实 online agent 更麻烦：用户或环境通常只给一个 prompt 一次 trajectory feedback，并不会同时提供 8 条 rollout 让你算组内均值。

所以论文把目标改成：

```text
one prompt -> one rollout -> train immediately
```

## 4. 方法拆解

### 4.1 总体框架

SAO 可以拆成四层：

1. Single rollout：每个 prompt 只生成一条轨迹，完成即进入训练。
2. DIS：直接比较 current policy 和 rollout policy 的 token probability，屏蔽过度 off-policy 的 token。
3. Stronger critic：critic 比 actor 更新更快，并冻结 attention 稳定 value training。
4. Skip-observation GAE：只沿模型 action 传播 advantage，跳过环境 observation。

### 4.2 关键公式：直接双边 Importance Sampling

异步训练很难精确保留每个历史 old policy。SAO 直接使用 rollout engine 当时记录的 log probability：

```text
r_t(theta) = exp(
  log pi_theta(a_t | s_t)
  - log pi_rollout(a_t | s_t)
)
```

然后定义：

```text
f(r_t) = r_t,  if 1 - eps_low < r_t < 1 + eps_high
         0,    otherwise
```

直觉是：

- 当前 policy 和行为 policy 足够接近，保留这个 token 的梯度。
- ratio 太极端，说明 token 已经过度 off-policy，直接 mask。
- 不再维护一大串历史 policy checkpoint 来重算精确 ratio。

这个方法接受少量可控 bias，换取更简单的异步实现和更稳定的 update。

### 4.3 Single Rollout 为什么还需要 Critic？

GRPO 用同一 prompt 的多条 reward 做 baseline；single rollout 没有组内对照，梯度方差会很高。

SAO 因此回到 value-based actor-critic，并做了三件事：

- 每次 actor 更新，critic 更新两次，让 value estimate 跟上快速变化的 policy。
- value model 冻结 attention，只训练 MoE projections，抑制过大的 critic gradient。
- 扩大 value pretraining data，缓解 single-rollout 训练开始时的 critic cold start。

这里的 know-how 是：single rollout 能否工作，很大程度取决于 critic 是否足够准，而不是只取决于 sampling 调度。

### 4.4 为什么 GAE 要跳过 Observation？

Agent trajectory 不是普通文本：

```text
action_0 -> observation_0 -> action_1 -> observation_1
```

`observation` 是环境返回，不是模型生成。如果 token-level GAE 沿 observation 传播，value model 会把外部文本也当成 policy action 的连续 token。

SAO 的做法是从 `action_i` 的最后一个 token 直接连到 `action_{i+1}` 的第一个 token：

```text
delta = reward
  + gamma * V(next action)
  - V(current action)
```

这样 advantage 只落在模型真正能控制的输出上。

## 5. 关键实验

| Claim | Evidence | My read |
|---|---|---|
| SAO 在长程 reasoning 上明显强于 GRPO | AIME2025: 97.3 vs 84.2；BeyondAIME: 74.8 vs 54.8 | 差距很大，但依赖 Qwen3-30B-A3B + TIR SFT 设置 |
| coding 上也有提升 | SWE-Bench Verified: base 23.0，GRPO+DIS 27.0，SAO 29.8 | coding 增益更温和，但方向一致 |
| DIS 先解决 collapse | 标准 GRPO 约 160 steps 后 collapse；GRPO+DIS 能稳定训练 | 严格 token trust region 是基础组件 |
| 完整 SAO 后期继续拉开 | SAO 与 GRPO+DIS 前期接近，约 400 steps 后分化 | single rollout + critic 设计提供额外收益 |
| critic 设计确实重要 | critic 一次更新使 BeyondAIME 74.8 降到 69.8；running mean 只有 55.3 | 不能把 value model 当可选附件 |
| token-level 优于 step-level | 400 steps 时 89.8/66.8，高于两种 step-level 方案 | 长程推理仍需要细粒度 credit assignment |

## 6. Know-how

- 系统调度和 RL objective 必须一起设计。把 rollout 改异步，但保留强同步假设的 group baseline，问题只会换个位置出现。
- rollout engine 要保存 token-level behavior log-probability，否则训练端无法做 SAO 的直接 importance correction。
- single rollout 省掉 group barrier，却引入更强的 critic 依赖；算力和显存账不能只算 actor。
- agent 轨迹里要显式区分 `action` 和 `observation`，credit assignment 不应该默认跨外部反馈 token 连续传播。
- 异步训练要监控的不只是 reward，还包括 policy lag、ratio distribution、masked-token ratio、critic explained variance 和 critic gradient norm。
- online feedback 天然是 single-trajectory，但把算法部署到真实用户流量前，还需要隐私、安全与灾难性遗忘机制。

## 7. Takeaways

面向研究：

- GRPO 的 group-relative estimator 不只是一个 loss 选择，它隐含了“同 prompt 多采样并等待整组”的数据到达假设。
- 在 agent RL 里，observation 是否由 policy 生成，应该进入 advantage estimator 的定义。
- single-rollout 的核心难点是 variance reduction；一个跟得上 policy 的 critic 重新变得重要。

面向工程：

- 长尾 rollout 下，先检查算法是否有 group barrier，再谈调度层异步化。
- 保存 rollout-time log-probability 和 policy version，是异步 RL 可训练性的基础数据合同。
- SAO 目前证明得更充分的是训练稳定性和任务效果，不是端到端系统吞吐。

一句话总结：

> 异步 Agent RL 不是把 rollout queue 解耦就结束了；sampling unit、off-policy correction、critic 和 action/observation credit assignment 必须一起重写。

## 8. 局限和我的判断

- 论文用“异步更高效”作为动机，却没有给 wall-clock、GPU utilization、throughput 和总训练成本对比。它证明了 optimization effectiveness，但系统效率账还没算完整。
- SAO 需要单独的 value model，而且 critic 每个 actor step 更新两次；这部分额外成本可能抵消一部分 group barrier 收益。
- SAO batch 有 128 个不同 prompts，GRPO batch 是 16 个 prompts × 8 rollouts。prompt diversity 和 estimator 同时变化，因果归因还不够干净。
- frozen attention 的结论来自 Qwen3 MoE critic，不一定直接适用于 dense model。
- GLM-5.2 的实际部署没有在正文给出详细实验，暂时不能把 750B-A40B 的应用描述当作完整 scale validation。
- online experiment 只是写作风格偏好切换，不代表真实用户环境中的长期在线学习已经解决。

我的判断：这篇论文最有价值的是指出了 GRPO 与异步/online agent 数据流之间的结构不匹配。SAO 的具体 critic 技巧可能随 backbone 演化，但“每条轨迹独立到达时，不能继续依赖组内同步 baseline”这个问题会长期存在。

## 9. 配图规划

| 图 | 内容 | 目的 |
|---|---|---|
| 1 | 封面：Agent RL 为什么不该等齐 8 条 Rollout？ | 建立问题 |
| 2 | 长短 trajectory 与同步 straggler | 解释异步动机 |
| 3 | GRPO group barrier vs SAO ready-then-train | 展示结构差异 |
| 4 | Policy lag：rollout policy vs current policy | 定义 off-policy 问题 |
| 5 | DIS ratio + 双边 mask 公式 | 讲稳定化核心 |
| 6 | Stronger critic：2× update + frozen attention | 解释 single-rollout 如何降方差 |
| 7 | Skip-observation GAE | 体现 agent trajectory 特性 |
| 8 | 五个 benchmark 结果 + collapse 曲线 | 支撑 claim |
| 9 | “论文证明了什么 / 还没证明什么” | 给出工程判断 |

## 10. 正文草稿

### 开头

为什么长程 Agent RL 不该继续等齐同一个 prompt 的 8 条 rollout？

因为 agent trajectory 的长度差异太大。短任务很快结束，长任务可能要跑几十甚至几百轮工具交互。同步 batch 会被最慢轨迹拖住；而 GRPO 即使放进异步系统，仍然要等同组多条 rollout 凑齐才能算 advantage。

### 方法

SAO 把训练单元改成 one prompt, one rollout。轨迹一完成就进入训练，不等同组样本。

但这样没有 GRPO 的组内 baseline，方差会变大。论文因此重新引入 value model：critic 每个 actor step 更新两次，冻结 attention 稳定梯度，并用更多 value pretraining data 解决 cold start。

同时，SAO 直接用 rollout 时保存的 token log-probability 计算 importance ratio。ratio 超出 trust region 的 token 不再裁到边界，而是直接 mask。对于 action 和 observation 交错的 agent 轨迹，它还让 GAE 跳过 observation，只沿模型自己的 action token 传播 advantage。

### 实验

结果很强：AIME2025 上 SAO 97.3，GRPO 84.2；BeyondAIME 上 74.8 vs 54.8；SWE-Bench Verified 上 29.8 vs 27.0。

更关键的是训练曲线。标准 GRPO 约 160 steps 后 collapse，加入 DIS 后稳定下来；完整 SAO 在约 400 steps 后继续拉开差距，并稳定跑到约 1000 steps。

### 我的 Takeaways

这篇论文提醒我们：RL algorithm 其实隐含了数据到达协议。

GRPO 假设同一个 prompt 可以同时采多条轨迹，并等待整组完成。真实 online agent 往往只有一次 trajectory feedback，异步系统也希望完成一条就消费一条。此时 group-relative baseline 从结构上就不匹配。

### 结尾

不过，SAO 还没有完整证明端到端系统效率。论文主要报告 accuracy 和 stability，没有报告 wall-clock、GPU utilization 或总训练成本，而且额外 critic 也不便宜。

所以更准确的结论是：SAO 给出了一个适配异步 agent 数据流、并且训练有效的 single-rollout 优化方案；它的实际系统性价比，还需要下一轮严格的效率对照。
