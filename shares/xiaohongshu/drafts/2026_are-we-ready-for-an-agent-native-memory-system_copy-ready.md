# Agent Memory 不只是 RAG：一篇 survey 讲清 4 个系统任务

> 图片顺序建议：
> 1. `../figures/2026_are-we-ready-for-an-agent-native-memory-system/01_figure2_representation.png`
> 2. `../figures/2026_are-we-ready-for-an-agent-native-memory-system/02_figure3_storage.png`
> 3. `../figures/2026_are-we-ready-for-an-agent-native-memory-system/03_figure4_extraction.png`
> 4. `../figures/2026_are-we-ready-for-an-agent-native-memory-system/04_figure5_retrieval.png`
> 5. `../figures/2026_are-we-ready-for-an-agent-native-memory-system/05_figure6_maintenance.png`
> 6. `../figures/2026_are-we-ready-for-an-agent-native-memory-system/00_table1_taxonomy.png`

## 正文

这篇论文不要当成“又一个 Agent Memory 方法”来看。

它更像一篇 survey + benchmark：作者把现有 agent memory system 拆成 4 个系统任务，然后把 12 个代表系统放到同一个 taxonomy 里比较。

我觉得这是一种很适合做论文分享的结构：不是一上来堆实验表，而是每个任务配一张范式图，再配代表系统和论文 citation。

核心框架是：

```text
Agent Memory System = <R, S, Q, U>
```

- `R`: Representation & Storage，记忆到底表示成什么、存在哪里。
- `S`: Memory Extraction，原始对话/工具日志怎么写成 memory object。
- `Q`: Retrieval & Routing，query 来了怎么找回相关 memory。
- `U`: Memory Maintenance，记忆变旧、冲突、过多时怎么更新和维护。

### 任务 1: Representation & Storage

第一件事不是“用不用向量库”，而是：agent 到底把经验记成什么？

论文把 representation 分成三类：

- Token-Level Sequence：把 memory 表示成文本序列、JSON memo、离散事实或内部状态。
- Graph & Tree-Based Topology：用图或树表达实体、关系、时间和层级摘要。
- Heterogeneous Composite：把 metadata、text、embedding、graph 等打包成复合 memory object。

对应的 storage 又分成三类：

- Transient In-Context Register：直接放在上下文或 KV cache 里。
- Specialized Single-Engine：放进单一后端，比如 vector DB、graph DB、relational DB。
- Heterogeneous Multi-Engine：多种后端一起用，比如 vector + graph + SQL / BM25。

原文 Table 1 里的代表系统：

- Token-Level Sequence：MemoChat [18]、Mem0 [5]、MEM1 [41]、MemAgent [35]。
- Graph / Tree：MemTree [27]、Zep [26]、Mem0g [5]、Cognee [21]。
- Heterogeneous Composite：LightMem [7]、SimpleMem [16]、MemOS [15]、MemoryOS [12]、A-MEM [33]、Letta [25]。

我的理解：Representation 决定 memory 的上限。你没有显式表示时间、实体和关系，后面 retrieval 再复杂，也很难稳定恢复这些结构。

### 任务 2: Memory Extraction

第二个问题是：原始经验流怎么写入 memory？

论文把 extraction 分成三类：

- Raw Sequence Concatenation：不做复杂抽取，直接拼接原始序列或递归摘要。
- Schema-Free Semantic Extraction：抽成自由文本事实或 embedding，没有强 schema。
- Schema-Constrained Structured Extraction：按预定义 schema 抽 entity-relation、typed payload 或结构化记录。

原文 Table 1 里的代表系统：

- Raw Sequence Concatenation：MEM1 [41]、MemAgent [35]。
- Schema-Free Extraction：Mem0 [5]、MemTree [27]、LightMem [7]。
- Schema-Constrained Extraction：MemoChat [18]、Zep [26]、Mem0g [5]、Cognee [21]、SimpleMem [16]、MemOS [15]、MemoryOS [12]、A-MEM [33]、Letta [25]。

这部分对工程最有启发：写入时不要过度过滤。

如果 extraction 太激进，很多未来 query 才会用到的线索会被提前删掉。论文后面的 ablation 也支持这个方向：write-time extraction 更应该保覆盖，query-time 再过滤。

### 任务 3: Retrieval & Routing

第三个问题是：query 来了，memory system 去哪里找证据？

论文把 retrieval / routing 分成五类：

- Native Attention-Based Retrieval：靠模型 attention 在上下文或内部状态里找。
- Semantic-Based Dense Retrieval：向量检索。
- Topological Subgraph Traversal：沿 graph / topology 做子图遍历。
- Autonomous Agentic Routing：让 LLM 自己生成 query plan 或 function call。
- Multi-Stage Hybrid Execution：多阶段混合检索，比如结构过滤 + dense search + BM25 + graph traversal + rerank。

原文 Table 1 里的代表系统：

- Native Attention-Based Retrieval：MEM1 [41]、MemAgent [35]。
- Semantic-Based Retrieval：Mem0 [5]、MemTree [27]、LightMem [7]。
- Topological Subgraph Traversal：Mem0g [5]、Cognee [21]、A-MEM [33]。
- Autonomous Agentic Routing：MemoChat [18]、SimpleMem [16]、Letta [25]。
- Multi-Stage Hybrid Execution：Zep [26]、MemOS [15]、MemoryOS [12]。

这部分最能说明：Agent Memory 不是 top-k embedding search。

比如问“她现在住在哪里”，你需要 current valid state；问“她什么时候开始养宠物”，你需要 temporal evidence；问“接着上次数据库操作做”，你需要 tool trace / procedural memory。

所以 retrieval 的关键不是多加一步 reasoning，而是 query 的约束能不能落到正确的 index、结构和证据粒度上。

### 任务 4: Memory Maintenance

第四个问题是长期系统最容易忽略的：memory 变旧、冲突、过多时怎么办？

论文把 maintenance 分成四类：

- Timestamp-Based Multi-Versioning：用时间戳、valid_from / valid_to、append-only log 等管理事实版本。
- Capacity-Driven Physical Eviction：按 FIFO、token limit、score / heat 等策略删除或替换。
- LLM-Driven Semantic Consolidation：用 LLM 合并、去重、摘要、mutation、pruning，或者 tool-driven CRUD。
- Continuous Parametric Optimization：通过离线训练或 fine-tuning 把历史反馈写进参数。

注意：Figure 6 画了 4 类 maintenance 方法；但原文 Table 1 的 12 个 evaluated systems 里，maintenance 列主要落在前三类。第四类在正文里作为方法范式出现，不作为 Table 1 里某个评测系统的主归类。

原文 Table 1 里的代表系统：

- Timestamp-Based Multi-Versioning：Zep [26]、Mem0g [5]、Cognee [21]、LightMem [7]、MemOS [15]。
- Capacity-Driven Physical Eviction：MEM1 [41]、MemAgent [35]、MemoryOS [12]、Letta [25]。
- LLM-Driven Semantic Consolidation：MemoChat [18]、Mem0 [5]、MemTree [27]、SimpleMem [16]、A-MEM [33]。

我觉得 maintenance 是 agent memory 真正走向生产的分水岭。

长期记忆最危险的不是“忘记”，而是旧事实还活着：用户已经搬到 London，系统还在回答 Paris；旧偏好已经失效，系统还继续引用；摘要把时间线压扁，后续 query 就再也找不回证据。

### 这篇论文最值得学的分享方法

我觉得这篇适合用一种“图 + 范式 + citation”的讲法：

1. 每张图只讲一个 memory task。
2. 每个 task 只保留 3-5 个范式。
3. 每个范式配原文 Table 1 里的代表系统。
4. 最后再讲 benchmark finding，而不是一开始就堆分数。

这样读者带走的不是“哪个系统排名第一”，而是一个架构检查表：

- 我的 memory 现在是 token、graph，还是 composite object？
- 写入时有没有过度 summarization？
- 检索是不是只有 embedding similarity？
- 更新和冲突有没有版本机制？
- 维护是局部更新，还是每次全局重组？

一句话总结：

Agent Memory 的核心难点不是“把历史存起来”，而是长期正确地表示、写入、检索和维护。

## 原文校验

以下只使用论文原文中明确出现的系统和引用号：

| 模块 | 原文位置 | 代表系统来源 |
|---|---|---|
| Representation & Storage | Figure 2, Figure 3, Table 1 | MemoChat [18], Mem0 [5], MEM1 [41], MemAgent [35], MemTree [27], Zep [26], Cognee [21], LightMem [7], SimpleMem [16], MemOS [15], MemoryOS [12], A-MEM [33], Letta [25] |
| Memory Extraction | Figure 4, Table 1 | MEM1 [41], MemAgent [35], Mem0 [5], MemTree [27], LightMem [7], MemoChat [18], Zep [26], Mem0g [5], Cognee [21], SimpleMem [16], MemOS [15], MemoryOS [12], A-MEM [33], Letta [25] |
| Retrieval & Routing | Figure 5, Table 1 | MEM1 [41], MemAgent [35], Mem0 [5], MemTree [27], LightMem [7], Mem0g [5], Cognee [21], A-MEM [33], MemoChat [18], SimpleMem [16], Letta [25], Zep [26], MemOS [15], MemoryOS [12] |
| Memory Maintenance | Figure 6, Table 1 | Zep [26], Mem0g [5], Cognee [21], LightMem [7], MemOS [15], MEM1 [41], MemAgent [35], MemoryOS [12], Letta [25], MemoChat [18], Mem0 [5], MemTree [27], SimpleMem [16], A-MEM [33] |

引用号沿用论文 References，不额外添加原文外 citation。
