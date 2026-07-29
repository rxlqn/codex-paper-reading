# ScientistOne: Towards Human-Level Autonomous Research via Chain-of-Evidence

- Year: 2026
- Date: 2026-05-25
- Venue: arXiv
- Authors: Rui Meng, Bhavana Dalvi Mishra, Jiefeng Chen, Chun-Liang Li, Palash Goyal, Mihir Parmar, Yiwen Song, Yale Song, Rajarishi Sinha, Parthasarathy Ranganathan, Burak Gokturk, Jinsung Yoon, Tomas Pfister
- Links:
  - Project: https://scientist-one.github.io/
  - arXiv: https://arxiv.org/abs/2605.26340
  - PDF: https://arxiv.org/pdf/2605.26340
  - Generated papers and solver code: https://github.com/scientist-one/generated-artifacts
- Local PDF: [../papers/2026_scientistone-towards-human-level-autonomous-research-via-chain-of-evidence.pdf](../papers/2026_scientistone-towards-human-level-autonomous-research-via-chain-of-evidence.pdf)
- Tags: agent, autonomous-research, chain-of-evidence, provenance, verification, multi-agent, literature-review
- Status: reading
- Rating:
- One-line takeaway: ScientistOne 把 autonomous research 的可信度问题从“论文写得像不像”改写为“每个 citation、number、method 和 conclusion 能否沿 typed provenance chain 回到文献、代码或实验日志”，并用 build-time grounding + post-hoc audit 同时约束研究生成与评测。

## Problem

自主研究 Agent 已经能完成文献调研、提出方案、执行实验和撰写论文，但常见评测主要看最终分数、论文外观或自动 review score，无法发现几类跨 artifact 的完整性错误：

- 引用条目根本不存在，或由模型参数记忆凭空生成；
- 论文中的 headline score 无法由提交代码和标准 evaluator 复现；
- 为了提高分数，solver 违反任务规范或利用 evaluator；
- method section 描述的算法和实际代码不是同一个方法；
- 多阶段 pipeline 在传递 summary 时把早期错误持续放大，最终形成一篇内部一致但外部不可验证的论文。

论文的核心问题不是“Agent 能否找到高分方案”，而是：一个自主研究系统应该保存什么证据结构，才能让论文中的 claim 被逐项核验？

## Core Idea

论文包含三个互相区分的贡献：

1. `Chain-of-Evidence (CoE)`：一个 architecture-agnostic 的研究可验证性标准。每个 claim 都必须通过记录下来的 evidence chain 追溯到 grounding source。
2. `ScientistOne`：一个按 CoE 原生构建的端到端自主研究系统，在 literature grounding、solution discovery 和 paper writing 三个阶段持续保存 provenance。
3. `CoE Integrity Audit`：一个适用于不同系统的事后审计协议，统一检查 score reproduction、specification violation、reference existence 和 method-code alignment。

核心设计原则可以概括成 `provenance before prose`：先形成带 evidence annotation 的结构化 research representation，验证 claim 与 source 的绑定，再生成最终 LaTeX；不是先写完论文，再尝试为文本补引用或解释。

## Method

### 1. CoE Standard

CoE 把当前较容易机器核验的 claim 分成四类：

| Claim type | Required evidence |
|---|---|
| Citation claim | 引用条目真实存在，且来源内容支持对它的描述 |
| Numerical claim | 数值能追溯到 evaluator output、实验日志或测量记录 |
| Methodological claim | 方法描述能定位到对应实现或实验记录 |
| Conclusion claim | 结论能由已验证的 numerical / methodological claims 推导 |

CoE 只定义“可验证”需要满足的性质，不强制规定系统架构。论文把它类比为数据库中的 ACID：标准描述可靠性边界，具体实现可以不同。

### 2. ScientistOne Pipeline

#### Stage 1: Literature Grounding

`Problem Investigator (PI)` 从 2-4 篇 seed papers 出发：

- 通过 Semantic Scholar citation / reference API 做最多两跳扩展，得到约 2,000-5,000 篇候选论文；
- 用 LLM 按 methodology relevance 和 problem alignment 评分，筛成约 500 篇 elite pool；
- 由 PI、Librarian、5 个并行 Researcher、SubdomainWriter 和 IslandConsolidator 做三轮调研，目标是阅读最多约 100 篇全文 PDF，形成 5-15 个 research directions；
- 对候选方向执行 evaluation-protocol audit，并对胜出方向补充 20-30 篇定向文献；
- 最终生成带 25-40 条可追溯引用的 experiment brief，包括研究版图、实验计划、baseline、metric 和 ablation 设计。

每条最终引用都来自实际 scholarly API 结果和被读取的 PDF，而不是写作阶段的模型记忆。

#### Stage 2: Discovery

- `Ideator` 从 experiment brief 生成保守方案和非常规方案，并按 novelty / feasibility 排序。
- `Parallel Explore-Exploit (PEE)` 在 `B` 个并行分支上搜索；每个 node 内 Solver 最多评测 `E` 个版本，每轮保留 top-`K` 分支，其余位置由精英方案的再 ideation 填充。
- 每个 Solver 在 sandbox 中迭代执行、调试和优化，并维护 experimental log。
- best-run selector 先过滤 specification violation，再选最高分方案，并由 ablation agent 分析关键组件。

论文的默认 search configuration 是 `I=5, B=5, K=2, E=4`，对应 25 个 search-tree nodes。

#### Stage 3: Paper Writing and Verification

Paper Writer 先在 Markdown research representation 上运行五步流程：

1. `Conceive`：把 PI brief、实验日志、verified scores、solver code、seed abstracts 组织成 narrative，并给每个 factual claim 绑定 evidence tag。
2. `Ground`：确定性检查 evidence tag、score、artifact path、baseline source 和 section completeness，把 claim 标成 supported / partial / unsupported。
3. `Critic`：用 LLM 检查 gap-approach alignment、矛盾、overclaim、缺失比较和 baseline fairness。
4. `Resolve`：删除或弱化 unsupported claims，修复矛盾；最多迭代两轮，grounding ratio 低于阈值时终止生成。
5. `Compose`：按 section 生成 LaTeX。

之后 `Claim Verifier` 再从成稿中抽取 claim，并按类型检查：

- 数值和日志在容差内是否一致；
- cite key 是否存在，以及论文摘要是否支持当前 assertion；
- method claim 与实验日志相应区域是否有实质性一致；
- malformed 或 `unsourced` claim 直接删除。

Refiner 根据 verifier 结果重写或删除问题句子。只有没有 blocking violation 的版本才输出为 final paper。

### 3. CoE Integrity Audit

Audit adapter 先把不同系统的 `paper.tex`、solution code 和 `references.bib` 统一成 artifact bundle，再执行四项检查：

- `I1 Score Verification`：从 TeX / PDF 抽取论文分数，用 golden evaluator 重跑代码，并在自适应容差内比较。
- `I2 Specification Violation`：LLM 同时读取 task spec、evaluator 和 solution code，多次独立判断后多数投票。
- `I3 Reference Verification`：用 Semantic Scholar、arXiv、OpenAlex 和 CrossRef 解析每条 bibliography entry，再用 LLM 消歧 near-miss。
- `I4 Method-Code Alignment`：LLM 对照 method section 与代码，多次独立判断后多数投票。

ScientistOne 还能做其他系统无法进行的 native check：`Claim Provenance Rate (CPR)` 衡量论文中的 quantitative claims 是否链接到数值一致的实验日志记录。

## Experiments

### Setup

- Primary benchmark: ADRS 的 5 个 systems-optimization tasks，分别是 Prism、Cloudcast、EPLB、LLM-SQL 和 TXN。
- Systems: Sakana AI-Scientist v2、AutoResearchClaw、DeepScientist、AI-Researcher、ScientistOne。
- Backbone: 所有被适配系统统一使用 Gemini 3.1 Pro 做 solver code generation 和 paper writing。
- Runs: 每个系统对每个任务跑 3 个 seeds，共生成并审计 `5 systems × 5 tasks × 3 seeds = 75 papers`。
- Score verification: evaluator 重复运行 5 次，用 `max(1%, 3σ / |mean score|)` 处理 evaluator variance。
- Baseline adaptation: 不同系统需要从 prompt-only 到修改 16-19 个源文件不等的 ADRS 适配；基础设施失败可以重新运行，但不能为了提高 solver score 重跑。

### Integrity Audit Results

| System | Score verification | Spec. violations | Hallucinated refs | Method-code aligned |
|---|---:|---:|---:|---:|
| Sakana AI-Scientist v2 | 5/12 | 10/15 | 0/159 | 5/15 |
| AutoResearchClaw | 5/12 | 0/15 | 3/196 | 3/15 |
| DeepScientist | 11/12 | 0/15 | 42/201 | 5/15 |
| AI-Researcher | 9/12 | 1/15 | 21/222 | 12/15 |
| ScientistOne | **12/12** | **0/15** | **0/337** | **14/15** |

EPLB 因为 score 含硬件相关执行时间，不进入 I1，所以每个系统的 I1 分母是 12。论文也提醒，Sakana 的 BFTS 与 ADRS artifact contract 不匹配，可能夸大其 I2 / I4 问题，因此不应把这些数字简单解释为绝对系统排名。

### Native Claim Provenance

- 15 篇 ScientistOne 论文中抽取出 639 个 numerical claims；
- 627 个通过 evidence matching，原始 CPR 为 98.1%；
- 12 个失败主要来自把硬件常量、LaTeX 下标或超参数误判为实验结果；
- 人工检查认为其中只有 2-4 个是真正 mismatch，对应修正后的 CPR 约为 99%。

### Paper Quality and Solver Performance

- ScholarPeer 自动 review 中，ScientistOne 的平均 accept rate 是 6/15，最佳 seed 选择后是 4/5；最强 baseline AI-Researcher 为 2/15。
- ScientistOne 的平均 overall score 为 4.5/10，best-of-3 为 6.6/10。所有系统的 clarity 都明显高于 soundness，说明“写得像论文”不是当前瓶颈。
- ADRS 上不同系统的 solver score 整体接近。ScientistOne 在 Cloudcast 和 EPLB 得到最佳 overall score，但 LLM-SQL、TXN 并非最优。
- 这支持论文最有价值的判断：当 solver performance 逐渐收敛时，系统差异更多来自实验结束后如何选择 artifact、绑定证据和写作，而不是谁能生成更流畅的论文。

### Generalization

论文还将同一 discovery loop 用于 5 个 MLE-Bench tasks 和 Parameter Golf：

- MLE-Bench 获得 2 个 Gold、2 个 Silver、1 个 Above Median；
- Parameter Golf 在 2026-04-27 的知识截止点上以 1.0600 达到当时榜首，并满足 16MB artifact 限制；
- 对比系统 DeepScientist 在 3D Object Detection 得分为 0，在 Parameter Golf 因超出大小限制形成 invalid submission。

这些结果主要证明 discovery loop 能跨任务运行；它们没有像 ADRS 一样完整验证每个领域中的 CoE audit 泛化。

### Search Scaling

附录比较了 PEE 的 width、depth 和 per-node evaluator budget，但每个 configuration 只跑一个 seed，且不同任务方差较大。作者因此把 search scaling 结果定位为 directional evidence，而不是稳定的 scaling law。

## Strengths

- 把“自主研究结果是否可信”拆成 claim、source、artifact 和 audit contract，而不是继续依赖最终分数或自动审稿印象。
- `provenance before prose` 很有工程价值：写作阶段消费已验证事实，能减少跨阶段 score cherry-picking、引用幻觉和 method drift。
- Audit 同时覆盖 paper、code、evaluator 和 bibliography，比只检查文字事实性更接近真实科研 artifact。
- 统一 backbone、3 seeds、canonical evaluator re-run 和 75-paper audit，使失败模式分析比少量 demo 更有说服力。
- 附录公开了大量失败案例，并明确承认 audit false negatives、baseline adaptation 和自动 review 的限制。
- 发布了 21 篇生成论文及对应 solver code，方便继续检查输出 artifact。

## Weaknesses

- “Towards Human-Level Autonomous Research” 的标题明显大于实验范围。ADRS 主要是带确定性 evaluator 的单指标算法优化，不能代表问题提出、实验设计、多数据集分析、理论论证或真实科学发现。
- I2 和 I4 依赖 LLM majority vote。I1-I3 的 flagged positives 由人类复核，但没有系统估计 false negatives；I4 只做了抽样人工验证。
- 附录承认某个 ScientistOne seed 的代码实际存在 evaluator exploit，但只有 1/5 个 I2 judges 标记，因此表格仍记为 `0/15`。这说明“零违规”是当前审计协议下的观测值，不是已证明的真实零失败。
- I3 主要验证引用是否存在，不充分验证引用内容是否支持当前 claim。Claim Verifier 只用 abstract entailment，仍可能漏掉 passage-level misattribution。
- ScientistOne 自身仍有 1/15 method-code misalignment；论文描述了不存在于代码中的 “hybrid neuro-symbolic solver” 和 “LLM-guided evolutionary search”。系统并未做到完全 by-construction correctness。
- 跨系统结果同时混入架构差异和 adaptation quality。四个 baseline 都不是为 ADRS 原生设计，Sakana 的高违规率尤其受 benchmark-interface mismatch 影响。
- 缺少关键的系统内消融：没有直接比较 ScientistOne 在关闭 PI grounding、evidence tags、Ground/Critic/Resolve 或 Claim Verifier 后的完整性变化，因此“优势来自哪一层”仍主要依赖机制解释和跨系统对比。
- 公开仓库提供的是生成论文和 solver code，而不是完整 ScientistOne pipeline；目前难以独立复现 75 次端到端运行和全部 audit。
- 成本信息不足：PI 每个 topic 最多阅读约 100 篇 PDF，PEE 默认搜索 25 个 nodes，但论文没有给出完整 token、API、算力、wall-clock 和人工复核成本。

## Useful For

- 设计 deep research / autonomous research 系统时，把 `claim -> evidence -> artifact` 作为一等数据结构，而不是只保存最终 summary。
- 给 Agent pipeline 定义 artifact contract：选中的 solver code、canonical score、ablation result 和 writeup context 必须指向同一 run。
- 为论文生成、实验报告或内部技术报告增加发布门禁：数值复跑、规范检查、引用解析、method-code 对齐。
- 将确定性检查与 LLM judgment 分层：路径、数值、cite key 和 evaluator output 尽量 deterministic；语义支持与方法对齐才交给 LLM，并显式记录 vote / confidence。
- 和 SearchOS 对照阅读：SearchOS 关注开放域搜索时 coverage 和 evidence state 如何构建；ScientistOne 关注研究全流程中 claim 如何绑定到 literature、code 和 logs。

## Questions

- CoE 应该只记录 claim 到 source 的单向引用，还是需要版本、冲突、supersession 和 source authority 等完整 provenance graph？
- 如何评估 I2 / I4 的 false-negative rate，而不是只人工复核被模型标记出来的 positives？
- conclusion claim 如何做可计算验证？如果结论依赖多个实验、统计检验和领域判断，typed evidence chain 需要什么 reasoning record？
- 如果没有 deterministic evaluator，实验日志、模拟结果、湿实验记录或证明草稿怎样形成可信 grounding source？
- build-time verifier 是否会让 Agent 只生成“容易验证”的保守 claim，从而牺牲真正有价值但难以形式化的新颖洞察？
- 能否对 PI、PEE、research representation、Claim Verifier 分别做关闭消融，量化每层对 correctness、paper quality、cost 和 latency 的贡献？
- CoE artifact 应该采用普通文件、数据库 schema、W3C PROV，还是 event-sourced evidence graph，才能支持跨 Agent、跨版本和长期审计？

## Share Notes

适合的分享主线：

1. 从 75 篇 Agent 论文的失败案例开场：高分、专业排版和可信研究是三件不同的事。
2. 解释 CoE 的四类 claim，以及每类 claim 应该落到什么 evidence。
3. 用一张流程图讲 ScientistOne：PI grounded literature -> PEE discovery -> provenance-first writer -> claim verifier。
4. 用 Audit 四项检查展示为什么只看自动 review score 不够。
5. 展示主结果时同时讲反例：ScientistOne 仍有 1 篇 method-code misalignment，I2 多数投票也漏掉一个已知 exploit。
6. 收束到工程 takeaway：最关键的不是增加更多 Agent，而是让每个 stage 的输出成为可寻址、可重放、可核验的 artifact。
