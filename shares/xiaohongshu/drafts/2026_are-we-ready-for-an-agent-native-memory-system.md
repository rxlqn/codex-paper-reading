# Agent Memory 不只是 RAG：这篇论文把它当成数据系统来评测

- Paper: Are We Ready For An Agent-Native Memory System?
- Note: [../../../notes/2026_are-we-ready-for-an-agent-native-memory-system.md](../../../notes/2026_are-we-ready-for-an-agent-native-memory-system.md)
- PDF: [../../../papers/2026_are-we-ready-for-an-agent-native-memory-system.pdf](../../../papers/2026_are-we-ready-for-an-agent-native-memory-system.pdf)
- Target audience: AI 硕博 / 大厂 AI 工程师
- Draft status: draft
- Suggested format: 7-9 张图 + 正文
- Keywords: agent memory, survey, benchmark, RAG, data management, long-term memory, evaluation

## 0. 一句话定位

这篇论文更像一篇 agent memory 的 survey + benchmark：它最值得讲的是，agent memory 不是“多加一个向量库”的 RAG 小改动，而是一个持久、可更新、可维护的数据管理系统。

## 1. 标题候选

- Agent Memory 不只是 RAG：这篇论文把它当成数据系统来评测
- 长期记忆怎么做才靠谱？12 个 Agent Memory 系统横向评测
- 为什么你的 Agent Memory 会记错、记旧、记乱？

## 2. 开头 Hook

很多 agent 项目做到 memory 时，会很自然地走向一个方案：把历史对话、工具结果、用户偏好塞进向量库，需要时检索回来。

但真实长程 agent 里，memory 不只是检索。

它还要处理：

- 新事实写入；
- 旧事实失效；
- 多版本冲突；
- 时间顺序；
- 检索路由；
- 成本和延迟；
- 长期运行后的维护。

这篇 survey 的价值，就是把 agent memory 从“RAG 组件”重新整理成“数据管理系统”的 taxonomy，并用系统评测验证不同架构的 trade-off。

## 3. 问题定义

- 输入: agent 运行过程中的 dialogue、tool execution logs、environment observations、user preferences、extracted facts。
- 输出: 未来某个 query / task 需要的 memory-augmented context。
- 系统目标: 在长期运行中正确保存、检索、更新、合并和淘汰 memory。
- 旧评测的问题: 只看最终 task score，把 memory system 当黑盒。
- 这篇论文的评测目标: 拆开 memory system 的结构，看不同模块如何影响效果、更新、稳定性和成本。

最小抽象：

```text
Agent Memory System = <R, S, Q, U>
```

解释：

- `R`: representation and storage，记忆如何表示、存在哪里。
- `S`: extraction，原始交互如何变成 memory object。
- `Q`: retrieval and routing，query 来了之后怎么找相关记忆。
- `U`: maintenance，如何处理冲突、遗忘、合并和容量。

## 4. 这篇怎么讲：4 个 memory task 范式

这篇最适合小红书的讲法不是复述实验表，而是把论文预定义的 4 个 lifecycle task 画出来：

```text
raw experience
  -> R: 怎么表示和存储？
  -> S: 怎么抽取成 memory object？
  -> Q: query 来了怎么检索/路由？
  -> U: 记忆变旧、冲突、过多时怎么维护？
```

每个 task 都可以做成一张“范式图”：左边是输入/失败模式，中间是 3-5 种代表范式，右边是适合的系统和 citation。

## 5. Task 1: Representation & Storage

### 图怎么画

标题：记忆不是一段文本，而是数据结构。

画四列：

```text
Token stream -> Vector / latent state -> Graph / tree -> Composite object
```

每列下面放一个具体例子：

- token stream: timestamped dialogue log / structured JSON memo；
- vector: fact embedding / KV-cache state；
- graph or tree: entity-relation-time graph、hierarchical memory tree；
- composite object: metadata + text + embedding + graph edges。

### 讲法

这一步回答的是：agent 到底把经验“记成什么”？

如果只记成文本，系统简单，但实体关系、时间变化和冲突很难显式表达。如果做成 graph / tree，结构更强，但索引和维护成本也会上来。Hybrid representation 更接近真实系统，但也更考验 routing 和 consistency。

### 可配 citation

| 范式 | 代表论文/系统 | 可以怎么引用 |
|---|---|---|
| Stream / text memory | MemoryBank, MemoChat | 长期对话记忆早期常用 stream + summary / memo |
| Tiered memory | MemGPT / Letta | 把 memory 分成 core / archival / recall 等层级 |
| Graph memory | Zep, Mem0 graph, Cognee | 用 temporal KG 或 entity-relation triplets 表示事实和变化 |
| Tree memory | MemTree | 用层级 schema 组织多轮对话和摘要 |
| Composite memory | A-MEM, MemOS, MemoryOS | 把 text、metadata、embedding、graph 或 index 组合成 memory object |

推荐引用：

- MemoryBank: Enhancing Large Language Models with Long-Term Memory, AAAI 2024.
- MemoChat: Tuning LLMs to Use Memos for Consistent Long-Range Open-Domain Conversation, 2023.
- MemGPT: Towards LLMs as Operating Systems, 2023.
- Zep: A Temporal Knowledge Graph Architecture for Agent Memory, 2025.
- A-MEM: Agentic Memory for LLM Agents, 2025.

一句判断：

> Representation 决定 memory 的上限：你没有显式表示时间和关系，后面检索再聪明也很难稳定恢复它们。

## 6. Task 2: Memory Extraction

### 图怎么画

标题：写入 memory 时，别太早把上下文过滤掉。

画三条 pipeline：

```text
Raw sequence concatenation
dialogue/tool log -> append as-is

Schema-free extraction
dialogue/tool log -> "User is vegetarian"

Schema-constrained extraction
dialogue/tool log -> {entity, relation, timestamp, source}
```

右侧标注三种风险：

- raw concat: 噪声多，后续检索压力大；
- schema-free: 粒度清楚，但容易丢上下文；
- schema-constrained: 可维护性强，但 schema 设计错了会系统性丢信息。

### 讲法

Memory extraction 解决的是“经验流如何变成 memory object”。这一步特别适合讲 know-how：不要在写入时过度总结。论文的 ablation 指向一个很实用的原则：write-time extraction 应该先保覆盖，再把过滤留给 query-time。

### 可配 citation

| 范式 | 代表论文/系统 | 可以怎么引用 |
|---|---|---|
| Raw sequence / summary | MEM1, MemAgent | 保留长程状态或递归摘要，减少外部解析 |
| Schema-free fact extraction | Mem0, LightMem | 从对话中抽独立事实，便于向量检索和更新 |
| Topic / memo extraction | MemoChat, MemOS | 把 conversation segment 成 topic / memo |
| Schema-constrained extraction | Zep, Cognee, Letta | 抽 entity-relation 或 typed payload，方便 graph / SQL / hybrid storage |

推荐引用：

- Mem0: Building Production-Ready AI Agents with Scalable Long-Term Memory, 2025.
- LightMem: Lightweight and Efficient Memory-Augmented Generation, 2025.
- MEM1: Learning to Synergize Memory and Reasoning for Efficient Long-Horizon Agents, 2025.
- MemAgent: Reshaping Long-Context LLM with Multi-Conv RL-based Memory Agent, 2025.
- Optimizing the Interface Between Knowledge Graphs and LLMs for Complex Reasoning, 2025.

一句判断：

> Extraction 的难点不是“抽得越干净越好”，而是不要把未来问题需要的线索在写入时提前删掉。

## 7. Task 3: Retrieval & Routing

### 图怎么画

标题：Memory 不是 top-k embedding search。

画五种检索路线：

```text
Native attention
Dense retrieval
Graph traversal
Agentic tool routing
Hybrid retrieval + reranking
```

每条路线配一个 query 示例：

- “她现在住在哪里？”需要 current valid state；
- “她什么时候开始养宠物？”需要 temporal evidence；
- “她喜欢什么餐厅？”可能需要 preference + recent context；
- “把上次数据库操作接着做完”需要 tool trace / procedural memory。

### 讲法

Retrieval/routing 回答的是：query 来时，系统该去哪一类 memory 里找？

这篇论文很适合用来反驳“向量库万能”。单纯 embedding similarity 对时间距离、冲突事实、多约束查询都不稳。更强的方式通常是 hybrid：先用结构过滤候选，再用 dense / BM25 / graph traversal 找证据，必要时让 LLM 生成 function call 或 query plan。

### 可配 citation

| 范式 | 代表论文/系统 | 可以怎么引用 |
|---|---|---|
| Dense retrieval | Mem0, LightMem, MemTree | 适合语义相近的事实回忆 |
| Graph traversal | Zep, Mem0 graph, A-MEM | 适合实体关系、邻域扩展、冲突事实 |
| Agentic routing | Letta, SimpleMem | LLM 先规划要查什么，再调 memory API |
| Hybrid retrieval | Zep, A-MEM, MemOS, MemoryOS | dense + sparse + graph + rule filters 的组合 |

推荐引用：

- Filtered Vector Search: State-of-the-art and Research Opportunities, VLDB 2025.
- BigVectorBench: Heterogeneous Data Embedding and Compound Queries are Essential in Evaluating Vector Databases, VLDB 2025.
- SimpleMem: Efficient Lifelong Memory for LLM Agents, 2026.
- MemOS: A Memory OS for AI System, 2025.
- Memory OS of AI Agent, EMNLP 2025.

一句判断：

> Retrieval 的关键不是多加一步 reasoning，而是让 query 的约束结构能落到正确的 index 和证据粒度上。

## 8. Task 4: Memory Maintenance

### 图怎么画

标题：长期记忆最大的问题不是忘，而是旧事实还活着。

画四种维护机制：

```text
Timestamp multi-versioning
Capacity eviction
Semantic consolidation
Tool-driven CRUD / parametric update
```

右边画 3 个典型事故：

- stale fact: “她已经搬到 London，但系统还说 Paris”；
- over-compression: “摘要把时间线合并没了”；
- delayed flush: “最近对话还没写入，检索不到”。

### 讲法

Maintenance 是这篇最值得工程同学关注的部分。只要 memory 是可写的，就一定会遇到更新、冲突、容量和压缩。论文的结论很实际：conservative consolidation 通常比过度总结或延迟写入更稳；localized maintenance 往往比 global reorganization 更划算。

### 可配 citation

| 范式 | 代表论文/系统 | 可以怎么引用 |
|---|---|---|
| Timestamp / multi-version | Zep, Mem0 graph, LightMem, MemOS | 用 validity / provenance / timestamp 管理事实变化 |
| Capacity eviction | MEM1, MemAgent, Letta, MemoryOS | 用 FIFO、token limit、heat score 或 queue flush 控制容量 |
| Semantic consolidation | MemoChat, MemTree, SimpleMem, A-MEM | 用 LLM 做合并、摘要、mutation、pruning |
| Tool-driven CRUD | Mem0, Letta | 让 LLM 通过 API 显式 create / update / delete memory |

推荐引用：

- Lifelong Learning of Large Language Model based Agents: A Roadmap, 2025.
- LifelongAgentBench: Evaluating LLM Agents as Lifelong Learners, 2025.
- Memory in the LLM Era: Modular Architectures and Strategies in a Unified Framework, VLDB 2026.
- Graph-based Agent Memory: Taxonomy, Techniques, and Applications, 2026.
- Memory in the Age of AI Agents, 2025.

一句判断：

> Maintenance 决定 memory 能不能长期上线：没有版本、冲突和压缩策略，memory 越多，错误越隐蔽。

## 9. 配图规划

| 图 | 内容 | 目的 |
|---|---|---|
| 1 | 封面：Agent Memory 不只是 RAG | 建立问题 |
| 2 | RAG vs Agent Memory：read-only retrieval vs persistent lifecycle | 区分边界 |
| 3 | 总框架：`M_sys = <R, S, Q, U>` | 给出主线 |
| 4 | Task 1: Representation & Storage 四种范式 | 讲“记成什么” |
| 5 | Task 2: Extraction 三种范式 | 讲“怎么写入” |
| 6 | Task 3: Retrieval/Routing 五种范式 | 讲“怎么找回来” |
| 7 | Task 4: Maintenance 四种范式 | 讲“怎么长期管理” |
| 8 | 12 个系统放到四任务矩阵里 | 体现 survey 价值 |
| 9 | 工程 checklist + citations | 输出可复用价值 |

## 10. 正文草稿

### 开头

这篇论文我觉得不能按“又一个 agent memory 方法”来读。它更像一篇 survey + benchmark，把 agent memory 拆成 4 个工程任务。

很多人说 memory，脑子里是一个向量库。但如果 agent 真的长期运行，memory 不是 top-k 检索，而是一个完整 lifecycle：

```text
表示/存储 -> 抽取 -> 检索/路由 -> 维护
```

### 四个任务

第一个任务是 representation and storage：经验到底记成 token、embedding、graph/tree，还是 composite object？这决定了后面能不能表达时间、关系和冲突。

第二个任务是 extraction：原始对话和工具日志怎么写成 memory？这里最容易犯的错是过度总结。写入时抽得太干净，未来 query 需要的线索可能已经没了。

第三个任务是 retrieval and routing：query 来了去哪里找？这一步不是简单 embedding top-k。时间、实体、关系、多约束查询，经常需要 graph traversal、structured filtering、query planning 或 hybrid retrieval。

第四个任务是 maintenance：记忆变旧、冲突、过多时怎么办？长期 memory 最危险的不是忘记，而是旧事实还被系统当成新事实。

### 为什么这个讲法适合分享

这篇论文的图特别适合改成小红书图文：每一张图只讲一个 task，每个 task 放 3-5 种范式，再配代表论文。

这样读者不会只记住“12 个系统谁分数高”，而是能带走一个架构检查表：

- 我的 memory 需要表达时间吗？
- 写入时有没有过度过滤？
- 检索是不是只有 embedding similarity？
- 更新和冲突有没有版本机制？
- 维护是局部的，还是每次都全局重组？

### 我的 takeaways

我最喜欢这篇的地方，是它把 agent memory 从一个“模型组件”重新放回到数据系统问题里。

真正上线的 agent memory，不是把历史存起来就结束了，而是要长期正确地写入、检索、更新和维护。

### 结尾

所以这篇适合当成 agent memory 的入门 survey 读。读完以后，不要先问“我要不要用 graph memory”，而是先问：我的 memory 系统现在卡在 `R/S/Q/U` 哪一个任务上？
