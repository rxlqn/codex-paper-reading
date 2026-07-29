# SearchOS：真正关键的不是多 Agent，而是给搜索系统一个显式状态

- Project: [antins-labs/SearchOS](https://github.com/antins-labs/SearchOS)
- Related note: [../../../notes/2026_searchos-structured-multi-agent-search-system.md](../../../notes/2026_searchos-structured-multi-agent-search-system.md)
- Reviewed revision: `fac992e` (2026-07-16)
- Target audience: AI 硕博 / 大厂 AI 工程师 / Agent 系统研发
- Draft status: draft
- Suggested format: 9 张图 + 正文
- Keywords: SearchOS, deep research, multi-agent, agentic search, SOCM, evidence grounding, context management

## 0. 一句话定位

SearchOS 最有价值的不是并行开 8 个搜索 Agent，而是把开放域问题编译成关系型 Coverage Map：每个 Agent 围绕缺失单元格工作，所有事实、来源、任务和进度都进入持久化状态，不再依赖聊天历史充当数据库。

## 1. 标题候选

- SearchOS 开源了：它把 Deep Research 做成了一个“搜索操作系统”
- 多 Agent 搜索为什么总在重复劳动？SearchOS 的答案是 Coverage Map
- Deep Research 的核心不只是会搜索，而是知道“还缺什么”
- 我看完 SearchOS 源码：真正关键的是把 Search State 从对话里拿出来
- SearchOS 源码拆解：Schema、Scheduler、Evidence Graph 如何连起来

## 2. 开头 Hook

现在很多 Deep Research 系统都有类似流程：

```text
planner 拆任务
多个 agent 并行搜索
writer 汇总答案
```

但长任务真正麻烦的地方不是“并行度不够”，而是：

- 搜过什么，只存在于几十轮聊天记录；
- 哪些实体还缺哪些字段，没有全局视图；
- 多个 Agent 会换个关键词重复搜同一件事；
- 页面、事实、引用和最后的文字混在一个 context 里；
- 到底什么时候算搜完，往往靠模型自己感觉。

SearchOS 最值得看的地方，是它把这些问题从 prompt 层下沉到了系统层。

## 3. 问题定义：搜索不是一次生成，而是补全一个状态空间

假设用户问：

```text
整理 2025 QS 每个学科 Top 5 大学，
同时给出学校总体排名、申请截止日期和申请费。
```

普通 Agent 看到的是一个长 prompt。

SearchOS 先把它变成关系型 schema：

```text
subject_rankings(Subject, University, SubjectRank)
universities(University, OverallRank, Deadline, Fee)
```

然后把搜索进度表示成 cell 状态：

```text
C(entity, attribute)
  = missing | filled | uncertain | hard_cell
```

系统的目标也随之改变：

```text
不是：直接生成最终答案
而是：持续选择高优先级 missing cells，直到 coverage 足够或预算耗尽
```

这一步是整个项目最核心的抽象。

## 4. 六步工作流

### 1. Explore：先找“信息地形”，不急着找每个答案

Explore Agent 先判断任务类型、实体进入结果集的条件、候选实体、canonical hub、可能的数据源和歧义。

它更像 scout：先回答“该搜谁、去哪里搜、怎么分工”，而不是自己把所有字段填完。

### 2. Schema：把用户问题编译成 Coverage Map

Orchestrator 创建一个或多个表，定义：

- primary key；
- attributes；
- 字段格式和语义；
- seed entities；
- foreign key / relation。

注意它不是照搬最终展示宽表，而是按实体粒度做规范化。这样同一个大学的申请截止日期只搜一次，不会因为它出现在四个学科榜单里就重复搜四次。

### 3. Dispatch：每个任务都绑定具体缺口

Frontier task 里不只有一句自然语言，还记录：

- target cells；
- priority；
- blocked_by；
- table id；
- skill；
- search budget；
- attempt 和 cooldown。

Scheduler 按优先级派工，还会在 dispatch 前重新检查：如果 peer Agent 已经填完这些 cell，这个排队任务就直接取消，不浪费一次模型调用。

### 4. Extract：Agent 只负责找对页面

Search Agent 不直接写共享状态。

页面经过统一 middleware：

```text
open / find / access skill
  -> Evidence Observation
  -> judge extraction
  -> grounding + dedup
  -> Evidence Graph
  -> Coverage Map
```

每个 Evidence Node 会带 value、source URL、source excerpt、entity、attribute、confidence 和 provenance。

搜索与抽取解耦后，不同 Agent 不需要各自发明字段格式和引用规则。

### 5. Assess：判断 Agent 有没有真的推进状态

SearchOS 的 loop sensor 不只查重复 query，还会看：

- 连续 bare search 却不打开页面；
- 同一 query 在滑动窗口重复；
- 多次工具调用后 coverage 没变；
- evidence node count 没变；
- extraction buffer 是否仍在处理中。

所以“工具返回了很多字”不再等于“Agent 有进展”。

### 6. Write：模型写解释，代码写严格表格

Writer Agent 不搜索网页，只读 SOCM。

它写每个 section 时必须提交真实存在的 evidence ids，否则工具直接拒绝。

最后：

- 解释和叙事由 writer 生成；
- coverage table 由代码确定性渲染；
- citation `[N]` 和 sources list 用同一个 URL map 生成。

这比在 prompt 里写一句“不要幻觉、记得引用”强得多。

## 5. SOCM 到底存了什么

SOCM 可以理解成搜索过程的系统状态：

| 模块 | 保存内容 |
|---|---|
| Frontier | 任务、优先级、依赖、状态、Agent report |
| Evidence Graph | 事实、来源、摘录、置信度、冲突关系 |
| Coverage Map | schema、实体、属性、cell 状态 |
| Strategy Memory | 有效策略、失败 query、坏来源 |
| Budget | query / time / iteration 消耗 |
| Writer Outline | section、正文、notes、引用 evidence |

当前实现把主状态写到 session workspace 的 `search_state.json`，并用文件锁做原子 read-modify-write。

这说明它首先是单机可恢复系统，而不是分布式数据库架构。这个边界很重要。

## 6. Context management 最值得借鉴的一点

SearchOS 不让完整网页永远留在模型上下文里。

一个 search episode 结束后，后续 prompt 只保留：

- 调过哪些工具、输入是什么；
- 新增了哪些 evidence ids；
- 哪些表增加了多少 filled cells。

原始工具输出退出 prompt，但事实和来源仍留在 SOCM / trajectory。

所以：

```text
context compression != state deletion
```

这是很多长程 Agent 系统容易混淆的边界。

## 7. 三层 Skill，不只是 prompt 文件

源码里有三类 skill：

- Access：针对具体网站 / API 的可执行访问能力；
- Strategy：ranking、enumeration、multi-hop 等搜索方法；
- Orchestrator：控制全局表结构与调度方法。

Access executor 不直接 import 到主进程，而是在独立 Python worker 运行，限制网络、文件、subprocess、内存、CPU 和输出大小。

但作者也很克制：Python audit hook 不是 VM。更准确的说法是“加固的独立 worker”，不是绝对安全沙箱。

## 8. Benchmark 怎么看

README 报告它在 WideSearch 和 GISA 的 F1 指标上领先列出的 baseline，尤其强调 enumeration recall。

不过我会加三个限定：

1. 表格用的是 max@3，也就是三次取最好，不是平均表现。
2. 仓库有 dataset、runner 和 scorer，但当前快照没有附带完整逐次 `eval_results`。
3. Technical report 还在 roadmap，论文也仍在准备。

因此现阶段更适合说：

> 这是一套实现非常完整、设计值得研究的 alpha 系统；benchmark claim 有代码框架支撑，但还需要完整实验材料和第三方复现。

## 9. 我认为最值得复用的 5 个设计

### 1. 先定义完成空间，再启动 Agent

尤其是清单、调研、尽调和竞品分析，先定义行、列、关系和 eligibility test。

### 2. Worker 返回 observation，不直接写共享状态

统一 intake 负责 schema binding、grounding、dedup 和原子提交。

### 3. Watchdog 看 state delta

监控 evidence、coverage、frontier 和 outline 是否前进，而不是只看工具调用次数。

### 4. 模型生成与确定性渲染分工

模型负责解释；代码负责严格 schema、表格、引用编号和 sources。

### 5. Search state 不应该只活在聊天历史里

长任务要能 snapshot、resume、replay，也要能让不同 Agent 读取同一份真实状态。

## 10. 局限和我的判断

它现在仍然很 alpha：

- 项目刚开源不久，代码还在高频变化；
- 没看到常规 tests 目录；
- README 和实际 skill 数量已有小幅漂移；
- 文件锁适合单机，不是多机调度底座；
- schema 和 judge extraction 仍然可能由 LLM 判断错误；
- 250+ site-specific access skills 的长期维护会很重。

但从 Agent 架构角度，我觉得 SearchOS 提出了一个比“再加一个 planner”更重要的问题：

> Deep Research 的真正瓶颈，会不会不是搜索能力，而是有没有一个独立于对话、可验证、可恢复、可调度的 search state？

## 11. 配图规划

| 图 | 内容 | 目的 |
|---|---|---|
| 1 | 封面：多 Agent 不是重点，Search State 才是 | 建立主判断 |
| 2 | 普通 Deep Research 的四类失败 | 解释为什么需要系统层设计 |
| 3 | Query -> relational schema -> coverage cells | 讲清核心抽象 |
| 4 | Explore / Schema / Dispatch / Extract / Assess / Write | 总体流程 |
| 5 | Frontier + Scheduler 示意 | 讲 priority、dependency、stale task |
| 6 | Evidence Intake pipeline | 讲搜索和抽取解耦 |
| 7 | SOCM 六类状态 | 讲共享状态与可恢复性 |
| 8 | Model prose vs deterministic table | 讲引用和输出边界 |
| 9 | 能确认 / 尚未证明 / 可复用设计 | 给出克制结论 |

## 12. 可直接发布正文

最近看了 SearchOS 的文档和核心代码。它表面上是一个多 Agent Deep Research 项目，但我觉得真正值得关注的不是“能并行跑多少 Agent”，而是它给搜索过程定义了一份独立于聊天历史的系统状态。

很多 Deep Research 系统的问题不是不会搜索，而是不知道自己还缺什么：搜过的页面埋在几十轮 tool message 里，多个 Agent 重复找同一个字段，最后什么时候停止也主要靠模型感觉。

SearchOS 先把问题编译成关系型 Coverage Map。实体是行，属性是列，每个 cell 有 missing、filled、uncertain、hard cell 等状态。Scheduler 不再派发一个模糊的“继续研究”，而是围绕具体缺失 cell 建立 Frontier task，记录 priority、dependency、target cells、skill 和 search budget。

更重要的是，Search Agent 不直接写共享状态。它只负责搜索、打开并定位正确页面；Evidence Extraction middleware 统一把页面变成带 source、excerpt、schema binding 和 confidence 的 Evidence Node，再原子更新 Evidence Graph 与 Coverage Map。

这样搜索和抽取被拆开了：不同 Agent 不再各自发明字段格式，也不需要靠长 summary 彼此同步。

它的 loop sensor 也很有意思。系统不只看 query 是否重复，还看多次工具调用之后 coverage 和 evidence node count 有没有变化。所以“Agent 调了很多工具、返回很多文本”不等于真的取得了进展。

最终写作也是混合式的：Writer Agent 负责解释和结构，但写 section 时必须提供真实 evidence id；严格的 coverage table、引用编号和 source list 则由代码确定性生成，而不是再让 LLM 重写一遍。

我最喜欢的是它对 context 的处理：旧网页工具输出可以退出 prompt，但事实、来源、任务和失败历史仍留在 SOCM。也就是说，context compression 不等于 state deletion。

当然它现在仍然是 alpha。README 的 benchmark 用 max@3，technical report 还没发布，仓库虽然有 datasets、runner 和 scorer，但当前快照没有附完整逐次 eval artifacts；主状态也是单机 JSON + 文件锁，不是分布式数据库方案。

但从架构上看，SearchOS 给了一个很有价值的方向：Deep Research 的分水岭可能不是 Agent 数量，而是有没有一个可验证、可恢复、可调度、带证据的 search state。
