# SearchOS: 把开放域检索变成可调度的结构化状态机

- Year: 2026
- Date: 2026-07-16
- Source type: open-source project / agent system implementation note
- Project: antins-labs/SearchOS
- Reviewed revision: [`fac992e`](https://github.com/antins-labs/SearchOS/commit/fac992e5ba5dec9902e09487b9d71d6095806585)
- Links:
  - Repository: https://github.com/antins-labs/SearchOS
  - README: https://github.com/antins-labs/SearchOS/blob/main/README.md
  - Chinese README: https://github.com/antins-labs/SearchOS/blob/main/docs/README.zh.md
  - Installation: https://github.com/antins-labs/SearchOS/blob/main/docs/installation.md
  - Configuration: https://github.com/antins-labs/SearchOS/blob/main/docs/configuration.md
- License: MIT
- Tags: agentic-search, multi-agent, deep-research, structured-retrieval, evidence-grounding, orchestration, context-management
- Status: read
- One-line takeaway: SearchOS 最有价值的设计不是“并行开很多搜索 Agent”，而是把开放域搜索编译成关系型 Coverage Map，用持久化 SOCM 统一管理任务、证据、覆盖率和写作状态，让 Agent 围绕明确缺口工作，而不是在对话历史里凭感觉继续搜。

## 先说结论

SearchOS 可以概括成一句话：

> 把一个开放域问题编译成待填充的关系型表格，再把每个空单元格当成调度目标。

它试图解决的不是“模型能不能搜到某个事实”，而是长程开放域检索里的四类系统问题：

1. 搜过什么、还缺什么只存在于聊天历史，压缩上下文后容易丢。
2. 多个 Agent 重复搜索同一实体或同一属性，却没有全局去重和依赖管理。
3. 搜索、阅读、抽取、记忆、写作全塞给一个 Agent，事实格式和引用很容易漂移。
4. 搜索过程缺少可检查的完成条件，常常在“看起来搜得差不多”时结束。

SearchOS 的回答是：不要把 conversation history 当数据库。把任务队列、Evidence Graph、Coverage Map、策略失败记忆和 writer outline 都写入系统状态，Agent 只消费自己当前需要的切片。

## 它不是什么

先把几个容易混在一起的概念拆开：

- 它不是简单的“多 Agent + web search”。多 Agent 只是执行层，Coverage Map 和 SOCM 才是控制层。
- 它不是传统 RAG。RAG 通常从已有语料中召回相关片段；SearchOS 还要主动发现实体、设计 schema、派发网页检索、判断覆盖缺口并持续补全。
- 它不是让每个 Agent 自己维护 memory。搜索 Agent 不直接写共享状态，证据写入由 middleware 接管。
- 它也不是完整的关系数据库。Coverage Map 用关系型 schema 组织任务和结果，但底层主状态仍是工作区中的 JSON 文件。

## 核心抽象：Citation-grounded Relational Schema Completion

对于一个需要收集多个实体、多个属性的问题，可以把目标写成：

```text
E = {e1, e2, ..., en}          # 实体集合
A = {a1, a2, ..., am}          # 属性集合
C(e, a) in {missing, filled, uncertain, hard_cell}
```

搜索系统的目标不再只是生成回答 `y`，而是持续选择尚未解决的 `(entity, attribute)`：

```text
while budget_available and exists missing cell:
  choose high-priority missing cells
  dispatch search tasks
  extract sourced evidence
  update coverage and conflicts
```

最后才从结构化状态生成用户答案：

```text
answer = render(coverage_map, evidence_graph, writer_outline)
```

这个重写非常关键：搜索的停止条件从“Agent 觉得够了”变成了一个可观测的 coverage、frontier 和 budget 联合判断。

## 一次查询的实际生命周期

根据 README、orchestrator prompt 和 session / lifecycle 代码，主流程可以整理为六步。

### 1. Explore：先摸清信息地形

除单实体、单属性的简单查询外，orchestrator prompt 要求先派一个 Explore Agent。

Explore 不负责逐项找答案，而是：

- 判断是 enumeration、fact lookup、comparison、trend 还是 open exploration；
- 明确实体进入结果集的 eligibility test；
- 通过多波并行检索寻找 canonical hub、候选实体和数据源分布；
- 建议表结构、主键和工作分片方式；
- 标记容易混淆的术语与应排除的 near-miss。

实现上，Explore Agent 与 Search Agent 是不同 prompt、不同职责。它在 schema 创建前工作，默认没有 Evidence Extraction middleware，因此它的主要产物是一份给 orchestrator 的 briefing，而不是直接填单元格。

相关实现：

- [`searchos/agents/explore/agent.md`](https://github.com/antins-labs/SearchOS/blob/fac992e5ba5dec9902e09487b9d71d6095806585/searchos/agents/explore/agent.md)
- [`searchos/agents/orchestrator/prompt.py`](https://github.com/antins-labs/SearchOS/blob/fac992e5ba5dec9902e09487b9d71d6095806585/searchos/agents/orchestrator/prompt.py)

### 2. Schema：把问题编译成关系型 Coverage Map

Orchestrator 调用 `create_schema`，把用户问题拆成一个或多个实体表。每张表包含：

- `table_id` / `table_label`；
- `primary_key`；
- `attributes`；
- `column_desc`，包括类型、语义、格式和机械校验；
- `seed_entities`；
- 表之间的 foreign key 和 relation。

这里不是简单照抄用户最终想看的宽表。prompt 明确要求按实体类型、主键和属性归属做规范化：如果大学总体排名属于 `University`，学科排名属于 `(Subject, University)`，就应该分成两张表，避免同一大学的主页、截止日期被重复搜索多次。

Coverage Map 支持：

- seeded closed table 与从零发现行的 column-only table；
- 多表和外键关系；
- composite primary key；
- entity / attribute alias；
- 新实体动态加入；
- soft delete 和恢复；
- 每个 cell 绑定多个 evidence id；
- `missing / filled / uncertain / hard_cell` 状态；
- 冲突证据标记。

相关实现：

- [`searchos/tools/schema.py`](https://github.com/antins-labs/SearchOS/blob/fac992e5ba5dec9902e09487b9d71d6095806585/searchos/tools/schema.py)
- [`searchos/socm/coverage.py`](https://github.com/antins-labs/SearchOS/blob/fac992e5ba5dec9902e09487b9d71d6095806585/searchos/socm/coverage.py)

### 3. Dispatch：围绕空 cell 建立 Frontier

搜索任务统一存进 `FrontierMemory`。一个 `FrontierTask` 不只有自然语言问题，还包含：

- `kind`: search / explore / write；
- `priority`；
- `blocked_by` 依赖；
- `target_cells`；
- `table_id`；
- `agent_type`；
- `skills`；
- 单任务 `max_searches`；
- attempt、状态、deadline、cooldown 和 report。

Scheduler 每次 tick 会：

1. 删除 `blocked_by` 环中的任务；
2. 解锁依赖已经结束的任务；
3. 回收失去运行 Agent 的 zombie task；
4. 检查证据冲突；
5. 视 coverage / evidence 增量启动或继续 writer；
6. 按 `(-priority, created_at)` 把 ready tasks 派到并发池。

它还会在真正 dispatch 前重新检查 target cells。如果其他 Agent 已经填完这些 cell，就直接把排队任务标成完成，避免过期任务继续消耗一次模型调用。

并发的核心并不是一次 `asyncio.gather`：系统还有全局 Agent 上限、spawn stagger、429 cooldown、依赖 DAG、stale-task revalidation 和 zombie recovery。这些才是“像 OS 调度进程”在代码里的具体含义。

相关实现：

- [`searchos/socm/frontier.py`](https://github.com/antins-labs/SearchOS/blob/fac992e5ba5dec9902e09487b9d71d6095806585/searchos/socm/frontier.py)
- [`searchos/tools/tasks.py`](https://github.com/antins-labs/SearchOS/blob/fac992e5ba5dec9902e09487b9d71d6095806585/searchos/tools/tasks.py)
- [`searchos/agents/orchestrator/scheduler.py`](https://github.com/antins-labs/SearchOS/blob/fac992e5ba5dec9902e09487b9d71d6095806585/searchos/agents/orchestrator/scheduler.py)

### 4. Search + Extract：搜索 Agent 找页面，middleware 写事实

Search Agent 的职责被刻意缩小：构造查询、打开页面、在页面内定位信息，最后写一段自然语言 briefing。它不直接操作 SOCM。

真正的 evidence intake 在 middleware：

```text
search/open/find/skill result
  -> EvidenceObservation
  -> buffer / deduplicate / merge same-page viewports
  -> judge extraction
  -> grounding and source excerpt checks
  -> EvidenceNode
  -> atomic update of Evidence Graph + Coverage Map
```

`EvidenceExtractionMiddleware` 只对 `open`、`find` 和 access-skill 结果触发抽取。普通页面默认缓冲处理，结构化 skill 结果同步处理；Agent 最终 summary 也会进入 intake，但被标为 `agent://...` 的派生来源。

`EvidenceIntake` 里有几个很工程化的细节：

- 用内容 hash 去重，失败时允许有限重试；
- 同一 URL 的多个 viewport 先合并，减少 judge 调用；
- 超长页面或大 JSON 会按结构 / 字符预算切片；
- 每张表有 flush lock，避免并发 extraction 用过期快照创建重复行；
- finalize 会等待后台抽取并冲刷残余 buffer，减少 Agent 被中断时的证据丢失；
- extraction pending 会反馈给 loop sensor，避免“judge 还没写完”被误判为无进展。

EvidenceNode 不只保存一句事实，还保存：

- `value` / `finding`；
- source URL 与 source excerpt；
- span / provenance；
- entity、attribute、table binding；
- confidence、alignment、source authority；
- active / rejected / superseded 状态。

Evidence Graph 的边可以是 support、conflict 或 refine。Coverage cell 则保留所有 supporting evidence ids，再按 provenance tier、alignment、source authority 和 confidence 选择主要展示值；不同值会显式触发 conflict，而不是静默覆盖。

相关实现：

- [`searchos/agents/search/agent.md`](https://github.com/antins-labs/SearchOS/blob/fac992e5ba5dec9902e09487b9d71d6095806585/searchos/agents/search/agent.md)
- [`searchos/harness/middleware/extraction/evidence_extraction.py`](https://github.com/antins-labs/SearchOS/blob/fac992e5ba5dec9902e09487b9d71d6095806585/searchos/harness/middleware/extraction/evidence_extraction.py)
- [`searchos/harness/middleware/extraction/intake/_engine.py`](https://github.com/antins-labs/SearchOS/blob/fac992e5ba5dec9902e09487b9d71d6095806585/searchos/harness/middleware/extraction/intake/_engine.py)
- [`searchos/socm/evidence.py`](https://github.com/antins-labs/SearchOS/blob/fac992e5ba5dec9902e09487b9d71d6095806585/searchos/socm/evidence.py)

### 5. Assess：用状态变化而不是文本长度判断进展

SearchOS 的 loop sensor 不是只有“同一个 query 搜了几次”。当前代码里至少包含五类信号：

1. 连续空结果 / 错误；
2. 连续 bare search 却不打开页面；
3. 滑动窗口中的重复 query；
4. 重复提醒后升级为 hard loop，写入 Agent 状态和 dead end；
5. 多次工具调用后 coverage 和 evidence node count 都没有变化。

检测到问题后，它优先注入策略提醒，不立刻粗暴杀掉运行；持续失败才把 Agent 标成 `looped`，由 orchestrator 决定换角度继续还是新派一个 Agent。失败 query 还会进入 strategy memory，避免后续 Agent 重走同一条路。

这体现了一个很好的系统原则：

> “工具返回了很多文本”不等于“检索取得了进展”；真正的进展应该落在状态增量上。

相关实现：

- [`searchos/harness/middleware/sensor/loop_sensor.py`](https://github.com/antins-labs/SearchOS/blob/fac992e5ba5dec9902e09487b9d71d6095806585/searchos/harness/middleware/sensor/loop_sensor.py)
- [`searchos/socm/strategy.py`](https://github.com/antins-labs/SearchOS/blob/fac992e5ba5dec9902e09487b9d71d6095806585/searchos/socm/strategy.py)

### 6. Synthesize：writer 写叙事，harness 确定性渲染表格和来源

Writer Agent 是长期存在的独立角色，不搜索网页。它读取 coverage summary、按实体 / 属性切片读取 evidence，并维护结构化 outline。

writer tools 强制两个约束：

- `write_section` 必须带 `cited_evidence_ids`；
- id 必须真实存在于 Evidence Graph，否则工具拒绝写入。

最终输出是“模型写作 + 确定性渲染”的混合：

- writer 负责解释、结构和叙事；
- Coverage table 由代码按 schema 和 evidence 生成，LLM 不重写；
- URL 到 `[N]` 的编号由代码统一分配；
- sources list 由同一个 citation map 生成。

这比单纯 prompt 要求“请确保每句话有引用”更强，因为结构化表格和来源编号不依赖模型最后一轮是否听话。

相关实现：

- [`searchos/agents/writer/agent.md`](https://github.com/antins-labs/SearchOS/blob/fac992e5ba5dec9902e09487b9d71d6095806585/searchos/agents/writer/agent.md)
- [`searchos/tools/writer.py`](https://github.com/antins-labs/SearchOS/blob/fac992e5ba5dec9902e09487b9d71d6095806585/searchos/tools/writer.py)
- [`searchos/harness/report/synthesis.py`](https://github.com/antins-labs/SearchOS/blob/fac992e5ba5dec9902e09487b9d71d6095806585/searchos/harness/report/synthesis.py)

## SOCM：系统状态到底包含什么

`SearchState` 是共享的 single source of truth，主要包含：

| State | 作用 |
|---|---|
| Frontier Memory | 搜索 / explore / write 任务、优先级、依赖、状态和 Agent report |
| Evidence Graph | 带来源、摘录、置信度、schema binding 和冲突关系的事实节点 |
| Coverage Map | 多表 schema、实体、属性、cell 状态和 evidence refs |
| Strategy Memory | 有效策略、失败 query、坏来源和 anti-pattern |
| Budget | query、时间、iteration 的消耗状态 |
| Writer Outline | section、正文、notes 和 cited evidence ids |
| Agent runtime flags | looped 状态、dead ends 等运行时信号 |

状态默认保存在 session workspace 的 `search_state.json`。`WorkspaceManager.atomic_update_state` 用文件锁包裹 read-modify-write，让并行 Agent 能看到彼此的更新并避免覆盖。完成的 turn 另存 snapshot，页面正文、metadata、trajectory 和最终输出也都在工作区落盘，因此可以恢复、重放和审计。

这里也要注意边界：当前持久层是文件系统 + `fcntl` lock，适合单机进程协作；它不是面向多机分布式 worker 的数据库事务方案。

相关实现：

- [`searchos/socm/state.py`](https://github.com/antins-labs/SearchOS/blob/fac992e5ba5dec9902e09487b9d71d6095806585/searchos/socm/state.py)
- [`searchos/socm/workspace.py`](https://github.com/antins-labs/SearchOS/blob/fac992e5ba5dec9902e09487b9d71d6095806585/searchos/socm/workspace.py)

## Context management：事实留在状态里，工具输出退出 prompt

SearchOS 的 context 管理不是简单把旧消息整体总结一次。

`SearchEpisodeMiddleware` 会以 search episode 为边界折叠历史：

- 保留工具调用的输入；
- 记录本 episode 新增的 evidence ids；
- 记录各表新增了多少 filled cells；
- 把原始工具输出从后续模型输入移除；
- 事实本体继续保存在 SOCM 和 trajectory 中。

这让模型看到的是“我做过哪些搜索、带来了哪些状态变化”，而不是不断携带整页网页文本。若 layered context 关闭，则退回按 token 预算临时裁剪消息的 `DynamicTrimMiddleware`。

相关实现：

- [`searchos/harness/middleware/context/layered_context.py`](https://github.com/antins-labs/SearchOS/blob/fac992e5ba5dec9902e09487b9d71d6095806585/searchos/harness/middleware/context/layered_context.py)
- [`searchos/harness/middleware/context/control_middleware.py`](https://github.com/antins-labs/SearchOS/blob/fac992e5ba5dec9902e09487b9d71d6095806585/searchos/harness/middleware/context/control_middleware.py)

## 三层 Skill 系统

当前 revision 的源码目录实际包含 253 个 access、31 个 strategy、1 个 orchestrator skill；README 的计数略有滞后，说明项目仍在快速变化。

三层职责不同：

| Layer | 形态 | 作用 |
|---|---|---|
| Access | site-specific `skill.md + manifest.yaml + executor.py` | 进入特定网站、API、登录墙或动态页面并返回结构化结果 |
| Strategy | 方法论 prompt / anti-pattern | ranking、enumeration、multi-hop、disambiguation 等搜索策略 |
| Orchestrator | 全局 playbook | 约束 orchestrator 如何设计表格或组织搜索 |

Access catalog 很大，所以会先用一个 LLM router 选 query-relevant top-k；失败时 fail-open，回退完整 catalog。每个搜索任务最多绑定少量 skills，避免把整个目录塞进 prompt。

Access Skill 的 executor 不在主进程直接 import，而是在独立 Python worker 运行。Execution Policy 可以限制：

- network 为 none / target hosts / any；
- 文件读写根目录；
- subprocess；
- native escape modules；
- CPU、内存、打开文件数、stdout 和结果大小；
- generated skill 只访问目标 host；
- bundled skill 可运行受识别的 Playwright driver。

作者也在代码注释里明确承认：Python audit hook 不是完整 VM。这里更准确的说法是“受策略约束的独立 worker”，不是强隔离容器。

相关实现：

- [`searchos/skills/catalog/router.py`](https://github.com/antins-labs/SearchOS/blob/fac992e5ba5dec9902e09487b9d71d6095806585/searchos/skills/catalog/router.py)
- [`searchos/skills/runtime/executor_runtime.py`](https://github.com/antins-labs/SearchOS/blob/fac992e5ba5dec9902e09487b9d71d6095806585/searchos/skills/runtime/executor_runtime.py)
- [`searchos/skills/runtime/_executor_worker.py`](https://github.com/antins-labs/SearchOS/blob/fac992e5ba5dec9902e09487b9d71d6095806585/searchos/skills/runtime/_executor_worker.py)

## 这套设计为什么有效

### 1. 把 completeness 变成一等公民

普通 deep research 更擅长找到“几个不错的答案”，但 enumeration 和 wide-table retrieval 要求的是尽量不漏。Coverage Map 把 recall 问题变成显式空格，scheduler 可以持续针对缺口派工。

### 2. 把并行搜索从“多份聊天”升级成共享状态协作

多个 Agent 不需要彼此转发长 summary。它们通过 Frontier 和 Evidence / Coverage 的原子更新协作，scheduler 还能在派工前取消已经被 peer 完成的 stale task。

### 3. 把搜索和抽取解耦

搜索 Agent 专注于落到正确页面，judge middleware 统一做 schema binding、单位 / 格式处理和 evidence 写入，降低不同 Agent 各自总结造成的格式漂移。

### 4. 把引用约束下沉到数据结构和工具层

EvidenceNode 带 source 与 excerpt，CoverageCell 只保存 evidence ref，writer tool 拒绝无证据 section，最终表格和 source list 又由代码确定性生成。Grounding 不再只靠一句 system prompt。

### 5. 把长上下文问题改成状态读取问题

原始网页和工具输出可以退出 prompt，只保留 episode 行为和状态 delta。Agent 需要事实时按表、实体、属性重新读取 SOCM，而不是依赖它“记得之前看过什么”。

## Benchmark：README 声称了什么，当前能确认什么

README 报告 SearchOS 在 WideSearch 和 GISA 的多项 F1 指标上优于列出的 baseline，其中 WideSearch Item F1 为 80.3，GISA Set F1 为 76.5；作者把主要收益归因于 recall-first 的 coverage-driven dispatch。

但这里需要严格区分：

- 可以确认：仓库包含 WideSearch / GISA 数据、评测 runner、benchmark adapter 和 scorer；相关命令与输出结构也已实现。
- 可以确认：核心代码确实实现了 README 所说的 schema、coverage、evidence、scheduler、writer 和 evaluation pipeline。
- 尚不能仅从这个仓库快照独立确认：README 汇总数字的每次原始运行配置、模型成本、方差和完整 replay artifact。仓库没有附带 README 所描述的逐 case `eval_results/`。
- 作者自己也把 technical report 放在 roadmap，并注明论文仍在准备中。
- README 使用 max@3，即三次运行取最好结果；比较时不能把它误读成平均性能或单次稳定性。

所以更稳妥的判断是：代码已经是一套相当完整的 alpha 系统，benchmark claim 有可执行评测框架支撑，但学术结论还应等待技术报告、完整运行材料和第三方复现。

## 工程成熟度判断

### 已经做得比较完整

- CLI、Textual TUI、Web UI、REST / WebSocket 都有实现路径；
- session、resume、follow-up、steering、stop、snapshot 与 replay 有明确状态边界；
- 多 provider、role-model binding、rate limit 和 effort tier 已配置化；
- schema / frontier / evidence / coverage / writer 的抽象不是 README 空壳，代码路径真实存在；
- 证据抽取、冲突、去重、stale task、zombie task、loop sensor、skill worker 都考虑了并发和失败场景；
- 对当前 revision 执行 `python3 -m compileall -q searchos eval` 通过。

### 仍然明显是 alpha

- `pyproject.toml` 明确标记 Development Status 为 Alpha，版本为 `0.1.0`；
- 项目 2026-07-07 才宣布开源，当前仍高频更新；
- README 与实际 skill 数量已经出现小幅漂移；
- 当前仓库没有常规 `tests/` / `test_*.py` 测试套件；
- 评测结果汇总存在，但完整原始运行 artifact 未随仓库提供；
- 主状态依赖单机文件锁，不是分布式执行底座；
- Judge extraction、schema 设计、entity alias 和 conflict arbitration 仍有 LLM 误判面；
- Access Skill 数量大，但站点变化、反爬、依赖和长期维护成本也会同步增长；
- Python audit hook + worker 是有价值的安全加固，但不能等同于容器 / VM 安全边界。

## 对其他 Agent 系统最值得复用的设计

### 1. 先定义完成空间，再启动 Agent

对于清单、调研、尽调、竞品、学术综述、数据收集类任务，先把“结果需要哪些行和列”显式化。Agent 的工作应该对应 coverage gap，而不是一个模糊大 prompt。

### 2. Worker 不直接写共享状态

让 worker 返回 observation，由统一 intake 做校验、去重和原子提交。这样 schema contract、provenance 和并发边界不会散落在每个 Agent prompt 里。

### 3. 用 state delta 做 watchdog

不要只看 Agent 是否在调用工具。检查它是否增加了 evidence、填了 cell、解决了 frontier task 或推进了 outline。

### 4. 模型生成与确定性生成分工

让模型写解释和综合，让代码渲染需要严格 schema、引用编号和格式稳定性的部分。

### 5. Context compression 不等于 state compression

可以从 prompt 删除原始工具输出，但不能删除可恢复的事实、来源、任务和失败历史。它们应进入独立持久层，并支持按需读取。

## 适用场景

SearchOS 特别适合：

- top-N × 多属性的宽表检索；
- “列出全部”或高召回实体枚举；
- 多来源、长时间、可中断恢复的研究任务；
- 需要逐 cell 引用和覆盖率可视化的分析；
- 数据分散在多个站点，需要站点专用 access skill 的任务。

不一定适合：

- 单实体、单事实查询；
- 主要依赖一个结构化数据库 API 的简单报表；
- 强实时、毫秒级响应；
- 需要严格分布式事务和多租户隔离的生产平台；
- 搜索目标无法合理表达成 entity / relation / attribute / sub-question 的开放创作任务。

## 我的判断

SearchOS 最有启发的不是又发明了一组 Agent 角色，而是把“搜索过程”从 prompt orchestration 提升成了一个数据与调度系统：

```text
query
  -> schema
  -> coverage gaps
  -> frontier tasks
  -> sourced evidence
  -> coverage updates
  -> deterministic table + cited writing
```

这条链路把任务完成度、事实来源、并行协作和恢复能力放在同一个 contract 里。对于 deep research 系统，真正的分水岭可能不是搜索 Agent 数量，而是有没有一个独立于聊天历史、可验证、可恢复、可调度的 search state。

## 后续值得继续验证的问题

- Technical report 发布后，SOCM 的正式定义和当前代码是否完全一致？
- max@3 之外，单次运行均值、方差、token / API 成本和 wall-clock 如何？
- Explore + Judge + Writer 多角色带来的成本，在哪类 query 上能被 recall 收益覆盖？
- Coverage Map 的 schema 若一开始设计错，系统如何低成本 migration，而不是不断 alias / edit？
- 多机 worker、共享数据库和任务队列化后，当前文件锁下的原子假设如何升级？
- Access Skill 自动生成如何做回归测试、版本化、权限审查和站点变化监控？
