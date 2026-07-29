# Agentic Search Systems

这个主题页整理 deep research、open-domain information seeking、multi-agent search、结构化检索状态、证据抽取与引用生成相关材料。重点不是“让模型多调用几次搜索工具”，而是搜索任务如何定义完成空间、共享状态、并发调度、证据协议和停止条件。

## 当前材料

### OfficeQA Pro: 把企业检索放回 parsing、版本与计算的完整链路

- 笔记: [notes/2026_officeqa-pro-enterprise-benchmark-for-end-to-end-grounded-reasoning.md](../notes/2026_officeqa-pro-enterprise-benchmark-for-end-to-end-grounded-reasoning.md)
- 论文: [arXiv:2603.08655](https://arxiv.org/abs/2603.08655)
- 代码与数据: [databricks/officeqa](https://github.com/databricks/officeqa)
- 关注点: 在近百年、89,000 页的 U.S. Treasury Bulletin corpus 上，端到端评测文档解析、跨文档 retrieval、temporal revision checking、外部检索和精确定量计算。
- 关键发现: 结构化解析让 frontier agents 提升 `6.0-20.3pp` 并加速 `4-9x`；标准 vector search 因丢失表头和页面语境而弱于 file search，contextual embeddings 与二者混合更有效。
- 核心判断: 企业 search 的 ground truth 不是一个 isolated chunk；它还包括文档时间、修订关系、table topology、统计口径与可重放的计算过程。

### ScientistOne: 从 evidence-grounded search 扩展到 claim-grounded research

- 笔记: [notes/2026_scientistone-towards-human-level-autonomous-research-via-chain-of-evidence.md](../notes/2026_scientistone-towards-human-level-autonomous-research-via-chain-of-evidence.md)
- 论文: [arXiv:2605.26340](https://arxiv.org/abs/2605.26340)
- 关注点: 用 Chain-of-Evidence 把 citation、numerical、methodological 和 conclusion claims 绑定到文献、代码与实验日志，并用统一 audit 检查跨 artifact 完整性。
- 关键机制: citation-graph literature grounding、parallel explore-exploit discovery、provenance-first research representation、Claim Verifier、score / spec / reference / method-code audit。
- 核心判断: 搜索证据不是终点；证据必须持续绑定到后续实验、代码选择和论文 claim，才能避免多阶段 Agent pipeline 生成“内部一致但外部不可验证”的结果。

### SearchOS: 把开放域检索变成可调度的结构化状态机

- 笔记: [notes/2026_searchos-structured-multi-agent-search-system.md](../notes/2026_searchos-structured-multi-agent-search-system.md)
- 项目: [antins-labs/SearchOS](https://github.com/antins-labs/SearchOS)
- 关注点: 将开放域问题编译为关系型 Coverage Map，用 Frontier、Evidence Graph、Strategy Memory 和 Writer Outline 组成持久化 SOCM。
- 关键机制: Explore scouting、multi-table schema、cell-targeted dispatch、middleware evidence intake、state-delta loop sensor、deterministic coverage rendering、three-layer skill system。
- 核心判断: 多 Agent 是执行层；可验证、可恢复、可调度的 search state 才是系统层。

## 分析框架

后续阅读同类系统时，优先拆开以下边界：

| 问题 | 要检查的实现 |
|---|---|
| 搜索目标如何表示？ | query / plan / schema / entity-attribute cells / graph |
| 谁定义“搜完了”？ | coverage、frontier、budget、stop sensor 还是模型主观判断 |
| Agent 如何协作？ | 互发 summary、共享 memory、事件总线或原子状态更新 |
| 搜索与抽取是否解耦？ | worker 自己总结，还是统一 evidence intake |
| 引用如何约束？ | prompt 约束、tool contract、evidence ref、确定性渲染 |
| 文档版本如何处理？ | publication date、observation date、preliminary / revised、supersession graph |
| 解析不确定性如何保留？ | OCR alternatives、cell coordinates、table topology、原图回查 |
| 长上下文如何处理？ | summary、trim、episode folding、按需读取持久状态 |
| 并发是否真实可控？ | priority、dependency、stale task、retry、cooldown、zombie recovery |
| 状态是否可恢复？ | snapshot、resume、replay、turn isolation、持久化 contract |
| Skill 是什么层？ | 搜索策略、站点访问能力、orchestrator playbook、执行权限 |
| 评测是否可信？ | benchmark scope、单次/均值/max@k、成本、原始运行 artifact |

## 和相邻主题的关系

- [Agent Memory Systems](agent-memory-systems.md): 关注长期记忆如何表示、检索、更新和评测；agentic search 更关注为了完成当前开放域任务而主动构建、填充和消费 search state。
- [Human-Agent Collaboration](human-agent-collaboration.md): 关注人如何给 Agent 提供约束和判断；agentic search 还需要定义机器侧的 coverage、evidence 与调度 contract。
- [Agentic Model Training](agentic-model-training.md): 关注如何训练更强的工具使用和长程 Agent；agentic search systems 关注推理时系统如何约束、并行和验证这些能力。

## 后续可补充材料

- Deep Research / Web research agent 的系统与论文对比。
- WideSearch、GISA、BrowseComp 等评测的任务差异。
- Table-as-Search、A-MapReduce、Web2BigTable 等 baseline 的状态表示。
- 搜索 schema 自动设计、动态 migration 和 entity resolution。
- 多机调度、数据库持久化和多租户隔离。
- Evidence conflict arbitration、source authority 和 temporal validity。
