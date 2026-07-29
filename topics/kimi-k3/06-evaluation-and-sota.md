# 专题 6：评测、SOTA 判断与复现边界

## 结论先行

截至 2026-07-28，Kimi K3 可以合理称为“开放权重综合 frontier 模型”或“开放模型 SOTA 候选”，特别是 agentic、coding、长程执行和 multimodal tool-use 的组合能力。

不建议无条件写成“当前所有维度第一的开源模型”。更准确的表述是：

> K3 是首个开放 3T-class 模型；在报告覆盖的综合评测中，它整体领先所比较的其他开放权重模型，并在多项 agent/coding benchmark 达到全模型第一，但总体仍落后于最强闭源模型，且不同 harness、effort 和工具配置限制了横向可比性。

## 1. 证据层级

### A. 官方统一主表

主表覆盖 reasoning、coding、agentic、vision，但数据来源混合：

- K3 官方运行；
- 竞争模型官方或 leaderboard 分数；
- Artificial Analysis / Vals AI 等第三方分数；
- 若干 in-house benchmark；
- 多种 agent harness。

它适合判断“能力覆盖面”，不适合当作完全统一协议的科学对照实验。

### B. 第三方 headline evaluations

报告截至 2026-07-23 汇总：

| 第三方评测 | K3 | 报告截点排名/位置 |
|---|---:|---|
| Artificial Analysis Intelligence Index v4.1 | 57.1 | #4 / 580 |
| Vals Index | 74.7 | #2 / 39 |
| WebDev Arena | 1,678 Elo | #1 / 99 |
| Text Arena | 1,486 Elo | #8 / 200 |
| Agent Arena | 9.1 | #4 / 37 |

这些结果支持 K3 属于 frontier，但也表明它不是所有综合榜单第一。

### C. 开放权重对手

主表中明确标记的开放对手是 GLM-5.2。K3 在所列 reasoning、coding、agentic 指标上大多显著领先。但“开放生态第一”的判断仍应检查：

- 是否遗漏同期模型；
- 是否使用相同 checkpoint；
- 是否采用相同 effort、harness 和 tool budget；
- leaderboard 是否在报告后更新。

因此 SOTA 应附日期。

## 2. K3 真正领先的地方

### Coding

- ProgramBench 77.8：表中第一；
- SWE-Marathon 42.0：表中第一；
- Terminal-Bench 2.1 88.3：仅低于 GPT-5.6 Sol 88.8；
- FrontierSWE 81.2：第二，低于 Claude Fable 5 86.6；
- DeepSWE 67.5：低于 GPT-5.6 Sol 73.0 和 Fable 5 70.0。

它更像“长程 coding 第一梯队”，而不是所有 coding benchmark 统治。

### Agentic

表中第一的代表项目：

- BrowseComp 91.2；
- DeepSearchQA 95.0 F1；
- ResearchRubrics 76.2；
- MCPMark-Verified 94.5；
- AutomationBench 30.8；
- SpreadsheetBench 2 34.8；
- τ³-Banking 33.4；
- Harvey Lab-AA 94.6。

仍落后的项目：

- GDPval-AA v2：1,686，低于 Fable 5 和 GPT-5.6 Sol；
- AA-Briefcase：1,548，低于 Fable 5；
- OfficeQA Pro、OSWorld 2.0、SaaS-Bench 等未领先。

### Reasoning 与 Vision

- GPQA Diamond 93.5，接近最强但不是第一；
- HLE-Full 43.5 / 56.0，明显低于 Fable 5；
- OmniDocBench 91.1，表中第一；
- Math-Vision 94.3 / 97.8，tool 后与 GPT-5.6 Sol 并列；
- ZeroBench-main 23.0 / 41.0，tool 带来显著提升，但仍低于 Fable 5 的 tool result。

K3 的强项更偏“模型 + 工具循环”，不等价于纯模型静态推理全部领先。

## 3. 五类可比性问题

### Harness

K3 常用 Kimi Code；Claude 模型常用 Claude Code；GPT 模型常用 Codex；Terminal-Bench 还混入 Terminus 2。Agent benchmark 测到的是：

```text
model × harness × prompt × tool policy × context management
```

不能把差异全部归因到 base model。

### Effort

K3 多数结果为 `max`，GPT-5.5 为 `xhigh`，其他模型也有各自 effort 定义。名称相似不代表 token budget 相同。

### Tools

HLE、MMMU-Pro、CharXiv、MathVision、ZeroBench 同时报告无工具/有工具。工具类型、调用预算和 sandbox 会改变问题定义。

### Fallback / Guard

Claude Fable 5 有潜在 fallback，GPT-5.6 Sol 有 cyberguard。某些 coding task 中 refusal/fallback 会影响均值，报告也承认 Fable 在 SWE-Marathon 的 35% tasks 发生 fallback。

### Time

Leaderboard 会漂移。FrontierSWE 截至 7 月 16 日，第三方 headline table 截至 7 月 23 日。任何“SOTA”笔记都应记录截点。

## 4. 成本效率

报告给出的代表性结论：

- Kimi Code Bench 2.0：比 Fable 5 低 4 分，但成本为其 38%；
- BrowseComp：91.2，约 $2.03/task，为 GPT-5.6 Sol 成本一半，远低于 Claude max；
- GDPval-AA v2：距 GPT-5.6 Sol 50 Elo 内，成本低 13%，比 Fable 5 便宜 2.6x；
- AA-Briefcase：分数第二，成本约为 Fable 5 一半。

成本表更接近实际模型选型，但仍受定价时间、API provider、缓存、effort 和 harness 影响。它不能替代自有 workload 的 total cost test。

## 5. Open-weight 还是 Open-source

官方 model card 使用 “open-weight”。K3 License 允许使用、修改、分发、微调和商业处理权重及相关软件，开放程度较高；但完整训练数据、预训练/后训练代码、全部环境和精确计算预算未随报告公开。

因此推荐用词：

- 技术传播：开放权重 frontier model；
- 与社区习惯兼容：开源模型，但首次出现时注明“更严格地说是 open-weight”；
- 复现讨论：不可完全复现的开放权重 release。

## 6. 最小独立验证计划

若要验证 K3 是否适合自己的工作，不必重跑整张榜单。可以做一个 4×3 matrix：

| Workload | low | high | max |
|---|---:|---:|---:|
| Repository coding | 成功率 / 成本 / wall time | 同左 | 同左 |
| Deep research | evidence coverage / citation correctness | 同左 | 同左 |
| Office artifact | rubric pass / visual defects | 同左 | 同左 |
| Multimodal tool-use | perception / tool gain / failure recovery | 同左 | 同左 |

并固定：

- 同一 harness；
- 同一 tool schema；
- 同一 max wall time；
- 同一 context policy；
- 同一 judge 或 deterministic verifier；
- 至少 3 次重复；
- 保存完整 trajectory 和成本。

## 最终判断

K3 的发布材料足以支持“开放权重综合能力的新 frontier”，尤其是 agentic workflow 和 long-horizon engineering。它还不足以证明：

- 每个 benchmark 都是开放模型第一；
- 2.8T 是能力领先的主要因果因素；
- 1M raw context 比 context compaction 更有效；
- 普通团队能以接近官方成本自部署；
- 官方主表中的所有差异都来自模型本体。

SOTA 是一个带日期、任务、harness 和预算的陈述，不是模型永久属性。
