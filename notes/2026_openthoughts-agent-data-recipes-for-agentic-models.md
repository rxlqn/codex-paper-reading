# OpenThoughts-Agent: Data Recipes for Agentic Models

- Year: 2026
- Date: 2026-06-23
- Venue: arXiv / alphaXiv
- Authors: Negin Raoof, Richard Zhuang, Marianna Nezhurina, Etash Guha, Atula Tejaswi, Ryan Marten, Charlie F. Ruan, Tyler Griggs, Alexander Glenn Shaw, Hritik Bansal, E. Kelly Buchanan, Artem Gazizov, Reinhard Heckel, Chinmay Hegde, Sankalp Jajee, Daanish Khazi, Emmanouil Koukoumidis, Xiangyi Li, Hange Liu, Shlok Natarajan, Harsh Raj, Nicholas Roberts, Ethan Shen, Nishad Singhi, Michael Siu, Ashima Suvarna, Hanwen Xing, Patrick Yubeaton, Robert Zhang, Leon Liangyu Chen, Xiaokun Chen, Steven Dillmann, Saadia Gabriel, Xunyi Jiang, Anurag Kashyap, Boxuan Li, Yein Park, Minh Pham, Sujay Sanghavi, Lin Shi, Ke Sun, Yixin Wang, Zhiwei Xu, Erica Zhang, Siyan Zhao, Wanjia Zhao, Jenia Jitsev, Alex Dimakis, Benjamin Feuer, Ludwig Schmidt
- Links:
  - alphaXiv: https://www.alphaxiv.org/abs/2606.24855
  - arXiv: https://arxiv.org/abs/2606.24855
  - PDF: https://arxiv.org/pdf/2606.24855
  - Project: https://openthoughts.ai
- Local PDF: [../papers/2026_openthoughts-agent-data-recipes-for-agentic-models.pdf](../papers/2026_openthoughts-agent-data-recipes-for-agentic-models.pdf)
- Tags: agent, data-curation, SFT, RL, tool-use, benchmark, open-data
- Status: reading
- Rating:
- One-line takeaway: 这篇论文把 agent 训练从“找一个更强 teacher”推进到“系统化设计 data recipe”：通过任务来源、混合、过滤、teacher、长轨迹筛选和 RL 数据源的 controlled ablation，构建 100K 条开放 agentic traces，并把 Qwen3-32B SFT 到七个 agent benchmark 平均 44.8%。

## Problem

强 agent 的公开训练 recipe 仍然很不透明。已有开放工作经常围绕单一 benchmark，例如 SWE-Bench 或 Terminal-Bench，导致研究者很难判断哪些数据选择能泛化到多种 agentic task。

这篇论文的问题不是再提出一个单点 agent benchmark，而是问：如果目标是训练 broadly capable agentic model，训练数据应该怎样被生成、混合、过滤和验证？

## Core Idea

论文提出 `OpenThoughts-Agent`，核心资产是一个开放的 agentic data curation pipeline。作者把 SFT 数据管线拆成六个阶段做 controlled ablation，再用实验结果组装 `OpenThoughts-Agent-v2` 100K agentic traces。

最关键的结论有四个：

1. task source 的选择影响最大，且不同来源对 SWE / terminal / OOD benchmark 的收益很不均匀；
2. 任务混合比单一 top source 更稳，Top-4 / Top-8 mix 能避免只优化某一类 benchmark；
3. teacher 不是越强越好，benchmark 表现最强的模型未必产生最适合 student 学习的轨迹；
4. 长轨迹过滤有效，保留更多 turns 的 execution traces 能提升下游 agent performance。

## Method

- SFT data object:
  - `(task description, agent trajectory)` pair。
  - 轨迹由 teacher model 在 terminus-2 harness 和 Daytona sandbox 里生成。
- Six-stage SFT pipeline:
  - Task sourcing: 比较 95 个 task generation strategies。
  - Task mixing: 按阶段 3.1 的排名组合 top-N sources。
  - Task augmentation: 测试约束、hardening、mixed augmentation 等改写方式。
  - Task filtering: 用 LLM response length / difficulty signal 等方式筛任务。
  - Teacher model: 比较 GLM 4.7、Kimi、GLM 5、GPT-5.3-Codex 等 teacher。
  - Agent rollout filtering: 过滤 timeout、subagent traces、少于 5 turns 的 traces。
- Final SFT recipe:
  - 使用 top task sources，包括 SWE-Smith、StackExchange SuperUser、StackExchange Tezos、IssueTasks。
  - 对 Tezos 等低唯一任务量来源做 synthetic task augmentation。
  - 用 GLM-4.7-AWQ 生成 agentic rollouts。
  - 保留至少 5 turns 的 traces。
  - 组成 100K 规模的 `OpenThoughts-Agent-v2`。
- RL study:
  - 在 8B scale 上比较不同 RL data sources。
  - 使用 RLOO 和 binary verifier reward。
  - 重点验证 SFT data recipe 和 RL data source 是否能组合。

## Experiments

- Core benchmarks:
  - SWE-Bench Verified-100.
  - OpenThoughts-TBLite.
  - Terminal-Bench 2.0.
- OOD benchmarks:
  - Aider Polyglot.
  - BFCL-Parity.
  - MedAgentBench.
  - GAIA-127.
  - FinanceAgent-Terminal.
- Main results:
  - `OpenThinkerAgent-32B` 在七个 benchmark 平均 44.8%。
  - 相比 `Nemotron-Terminal-32B` 的 40.9%，提升 3.9 percentage points。
  - SWE-Bench Verified 54.0%，Terminal-Bench 2.0 26.2%。
  - 100K SFT 数据在 compute-controlled scaling 对比中优于其他 open datasets。
- Ablations:
  - task source 的性能跨度最大；
  - top source 混合优于只用 Top-1；
  - 常见 task augmentation 不一定提升；
  - LLM-based difficulty / response-length filter 有帮助；
  - 最强 benchmark model 不一定是最佳 teacher；
  - `min turns >= 5` 的 trajectory filter 在 matched token budget 下仍有收益。

## Strengths

- 把 agent data curation 从经验 recipe 变成可比较的 ablation study。
- 覆盖 SFT 和 RL 两条路径，不只讨论单阶段 post-training。
- 公开数据、pipeline、实验数据和模型，对开放 agent 训练研究很有价值。
- 对“teacher strength”和“trajectory learnability”的区分很重要：更强 teacher 不等于更好的 training data。

## Weaknesses

- SFT 部分主要固定在 Qwen3 family，base model 与 recipe 的交互没有完全隔离。
- RL 主实验只做到 8B scale，是否能迁移到 32B 还需要验证。
- 很多数据源和 harness 选择带有强工程依赖，复现不只是下载数据，还要复现 sandbox、verifier、评测环境。
- synthetic augmentation 的收益需要结合具体任务来源理解，不能泛化为“任意改写都会变好”。

## Useful For

- 设计 agent SFT 数据管线时，用作 checklist：source、mix、filter、teacher、trajectory length、scaling path。
- 对比 `Scaling the Horizon`：两者都强调长程轨迹，但这篇更像开放数据 recipe 和 ablation atlas。
- 对比 `Privileged Information Distillation`：那篇关注 teacher/student information gap，这篇关注生成可学习 agent trajectories 的数据工程。
- 做小红书 / 组会分享时，适合讲“agent 训练里真正重要的数据变量是什么”。

## Questions

- 为什么 GLM-4.7-AWQ 作为 teacher 比更强 benchmark model 更适合？是轨迹风格、verbosity、tool-use policy，还是 distribution match？
- `min turns >= 5` 的收益是否来自更完整的问题解决过程，还是来自筛掉了简单但低价值样本？
- synthetic task augmentation 对 Tezos 有效，是否因为它原始 unique tasks 太少？换到更大更多样的 source 是否仍有效？
- RL 的 `pymethods2test` 为什么能同时提升 core 和 OOD？是 verifier 干净、难度适中，还是训练行为更可迁移？
- 如果要复现到自己的 agent 系统，最小可行 pipeline 应该保留哪几个 stage？

## Share Notes

适合的分享主线：

1. 背景：开放 agent 模型缺的不是 benchmark，而是可复现的数据 recipe。
2. 问题：训练数据不是一个 blob，而是 task source、mix、filter、teacher、trajectory 的组合决策。
3. 方法：六阶段 SFT pipeline + 8B RL data-source study。
4. 关键发现：source matters、mix matters、stronger teacher may be worse、longer traces help。
5. 讨论：agent 数据工程的核心不是“多拿数据”，而是构造可学习、可验证、可泛化的 action trajectories。
